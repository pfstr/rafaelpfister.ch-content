---
title: "Testing Cold Storage with Rclone: A Practical Test Plan"
navTitle: "Testing Rclone"
description: "Before a service reads its files from the cloud through an Rclone mount, you should verify more than directory access. This test plan covers cold reads, warm reads, write operations, cache behavior, file integrity, and failures."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min read"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
translationOf: "cloud-mount-testen-dummy-pdfs"
slug: "testing-cloud-mounts-with-generated-pdfs"
translationId: article-8592f808b2e93cd4
translatedAt: 2026-09-03T08:18:04.922Z
translationReview: automatic
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/en/blog/testing-cloud-mounts-with-generated-pdfs
translationModel: gpt-5.6-terra
---

An Rclone mount is quick to set up. The remote appears as a directory, `ls` shows files, and the first functional test passes. That says little about production use.

As soon as a service accesses the mount, further questions arise: How long does the first access to a file take? Which accesses does the local cache serve? What happens to a file that has not yet been uploaded if Rclone crashes? Does a running container see the newly created mount again? And how does the service react if the cloud is temporarily unavailable?

This article provides a general test plan for that purpose. You can use it for a document archive, media server, photo management system, or any other service that retrieves infrequently needed files from cold storage through Rclone.

## The most important rclone options

For reference, here are the Rclone options used in this test plan, translated approximately from the documentation:

<details class="options-details">
<summary>Options at a glance</summary>

| Option | Meaning |
|---|---|
| `--seed n` | Initial value for the random number generator in `rclone test makefiles`; the same seed produces the same file tree |
| `--files n` | Number of test files to generate |
| `--files-per-directory n` | Average number of files per directory |
| `--min-file-size grösse` | Smallest generated file size (suffixes such as K, M, and G are allowed) |
| `--max-file-size grösse` | Largest generated file size |
| `--progress` | Continuous progress display during transfer |
| `--stats dauer` | Interval at which transfer statistics are output |
| `--log-file datei` | Writes the log to the specified file |
| `--log-level stufe` | Log detail level: DEBUG, INFO, NOTICE, or ERROR |
| `--one-way` | With `rclone check`, checks only whether all source files are present and identical at the destination; additional files at the destination do not count as errors |
| `--download` | Actually downloads the data during comparison instead of comparing hashes only |
| `--vfs-cache-mode modus` | Cache strategy of the VFS layer; `full` buffers read and write access locally |
| `--cache-dir verzeichnis` | Directory for the local cache |
| `--vfs-cache-max-size grösse` | Soft limit for the total size of the VFS cache |
| `--vfs-cache-poll-interval dauer` | Interval at which Rclone checks and cleans up the cache |
| `--vfs-write-back dauer` | Wait time after closing a file before uploading to the remote begins |
| `--vfs-read-ahead grösse` | Additional amount of data read ahead beyond the requested position with `full` |
| `--poll-interval dauer` | Interval at which Rclone polls the remote for changes (only for backends with polling support) |
| `--dir-cache-time dauer` | Validity period for cached directory listings |
| `--allow-other` | Allows users other than the one mounting it to access the FUSE mount |

</details>

The complete lists are available in the Rclone documentation, especially under [rclone mount](https://rclone.org/commands/rclone_mount/) and in the overview of [global flags](https://rclone.org/flags/).

## First define what you want to achieve

Cold storage does not automatically mean the same thing for every application. A media server usually reads large files sequentially. A photo management system loads many small preview files and jumps to different positions. A document archive opens comparatively small files, but often only once.

Before testing, note the key characteristics of your actual data set:

- typical file size and largest file encountered
- number of files per directory
- full reads or random access to individual ranges
- ratio of read to write access
- number of simultaneous users or processes
- changes that occur directly in the remote outside the mount
- acceptable wait time for a cold read
- maximum available space for the local cache

Only then can meaningful success criteria be established. Opening a single file in 1.2 seconds may be perfectly acceptable for an archive and unusable for an interactive application.

## Create a reproducible test data set

Rclone already includes a suitable tool for this. `rclone test makefiles` generates the same file tree every time with a fixed seed:

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
| `--seed 42` | Fixed initial value for the random number generator; every run creates the same data set |
| `--files 250` | 250 test files in total |
| `--files-per-directory 25` | An average of 25 files per directory |
| `--min-file-size 16K` | Smallest file: 16 KiB |
| `--max-file-size 32M` | Largest file: 32 MiB |

</details>

Adjust the number and sizes to match your actual data set. Do not test only average files. Some very small files show how expensive metadata access is; some large files reveal throughput, read-ahead, and cache behavior.

Also add file names that can cause problems in practice:

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
| `printf '…\n' > datei` | Writes the specified text as file content; redirection creates the file with the problematic name |

</details>

The last test is particularly important if the local file system and cloud backend handle uppercase and lowercase letters differently.

If your service accepts only certain formats, arbitrary binary files are not enough. In that case, also create synthetic files in precisely those formats. With Paperless-ngx, these were PDFs with a real text layer so the test would not accidentally measure OCR performance instead of the storage path. A photo management system should include different image sizes and formats, while a media server should include short files with different codecs.

## A baseline measurement without a mount

Before FUSE and the VFS cache come into play, measure the backend directly. Copy the data set to the test remote with Rclone and save a detailed log:

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
| `./testdata` | Copy source: the locally generated test data set |
| `remote:cold-storage-test` | Destination: path in the configured remote |
| `--progress` | Continuous progress display in the terminal |
| `--stats 5s` | Transfer statistics every 5 seconds instead of at the default interval |
| `--log-file rclone-copy.log` | Complete log in a file for later analysis |
| `--log-level INFO` | Logs transfers, retries, and errors without DEBUG-level volume |

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
| `./testdata` | Reference: the local original data set |
| `remote:cold-storage-test` | Item under test: the newly uploaded data set in the remote |
| `--one-way` | Checks only whether all source files are present and identical at the destination; additional files at the destination are not reported |
| `--download` | Downloads the data and compares content instead of relying on hashes |

</details>

`--download` is crucial here because some backends do not provide suitable hashes. The comparison takes longer, but provides a useful baseline for the later integrity test.

Record upload time, transfer rate, number of retries, and API errors. If direct access is already unstable, the mount cannot fix it.

## Separate the test mount from the production cache

Use a separate mount point and a separate cache directory for measurements:

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
| `--vfs-cache-mode full` | Fully buffers read and write access in the local cache |
| `--cache-dir /var/cache/rclone-test` | Separate cache directory, isolated from the production cache |
| `--vfs-cache-max-size 10G` | Soft limit of 10 GiB for the VFS cache |
| `--vfs-cache-poll-interval 1m` | Cache checks and cleanup once per minute |
| `--allow-other` | Users other than the one mounting it may also access it; necessary for services and containers |
| `--log-file /var/log/rclone-test.log` | Log to a file to trace access during testing |
| `--log-level INFO` | Medium log detail level |

</details>

These values are an example, not a general recommendation. The separation is what matters: an empty test cache makes cold reads reproducible without requiring you to delete files from an active production cache.

`--vfs-cache-mode full` is usually the most informative test mode for applications. Rclone buffers read and write access locally and can better emulate file access patterns that would not be possible with pure object storage. The added compatibility costs local storage space.

## Always test from the real service's perspective

A mount may work for your user yet still be unusable for the service. Common causes include a different user ID, missing `--allow-other`, container boundaries, or incorrect mount propagation.

Therefore, perform at least one full read using the same identity under which the application will later run:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-u <service-user>` | Runs the command as the specified user rather than root |
| `/mnt/rclone-test/pfad/zur/datei` | File to read; `sha256sum` forces a full read |

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

An actual application test is even better. Open the file through the service's web interface or API. Only then will you notice whether the application, for example, starts multiple parallel reads, jumps to the end of the file, or expects additional metadata.

## Measure cold reads and warm reads separately

With `--vfs-cache-mode full`, there are three layers between the application and the cloud:

| Layer | What it contains |
|---|---|
| Remote | The complete file in the cloud service |
| VFS cache | Locally stored portions of files that have already been read |
| Linux page cache | Recently used data in RAM |

For a cold read, choose a file whose contents have never been read through the test mount. During the immediately following warm read, it is in the VFS cache and usually also in RAM.

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
| `cat "$file" > /dev/null` | Reads the file completely and discards the output; only read time is measured |
| `"$1"`, `"$2"` | Shell function arguments: file path and measurement-line label |

</details>

Do not measure just one file. Use at least ten previously unread files of different sizes and record the median, slowest value, and file size. A single best result is not a basis for a decision.

A warm read is not a pure disk test because the kernel can retain parts of the file in RAM. For most cold-storage scenarios, that is not a problem. What matters is what a user experiences when opening a file for the first time and again later. If you want to evaluate RAM and local disk separately, you must additionally control and demonstrably clear the page cache.

## Test more than full reads

`cat` reads a file from beginning to end. Many applications behave differently:

- A video player initially reads headers and the index, later jumps to another position, and then continues loading sequentially.
- An image management system reads metadata and then creates a thumbnail.
- An archiving program may read the end of the file first.
- Multiple workers may access different files simultaneously.

Test these workflows with the actual application. Monitor the Rclone log and cache at the same time. For large files, it is interesting to see how much Rclone actually stores locally and whether `--vfs-read-ahead` fits the access pattern.

An Rclone mount is also not a sensible storage location for databases or other files that require reliable locking and frequent changes within the same file. The VFS layer compensates for differences between a file system and object storage, but it does not turn the backend into a local file system.

## Validate the write path separately

If your service only reads, mount the remote read-only whenever possible. If it must write, test creating, overwriting, renaming, and deleting separately.

A written file does not necessarily appear in the remote immediately. With an active VFS cache, uploading begins only after the file has been closed and `--vfs-write-back` has elapsed. Therefore, check both states:

1. The application has successfully closed the file.
2. The file can then be read in the remote through direct Rclone access.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# After --vfs-write-back has elapsed:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Target file in the mount; redirection writes through the VFS cache |
| `remote:cold-storage-test/writeback-test.txt` | Direct access bypassing the mount: `rclone cat` reads the file from the remote and outputs it to stdout |

</details>

Repeat the test with a large file and stop Rclone while the upload is still in progress. Then restart using the same cache directory and verify whether the upload resumes. This exact time window determines how much data is at risk during a server failure.

Also test renaming and deletion. Many cloud backends map these operations differently than a local file system. What matters is not only whether the command finishes successfully, but when the change becomes visible through direct remote access and to other clients.

## Test changes outside the mount

Files can be changed through the provider's web interface, a second Rclone process, or another server. The mount does not always see such changes immediately because directory information is cached.

Therefore, use a second Rclone invocation to create a file directly in the remote:

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
| `remote:cold-storage-test/external-change.txt` | Destination with the exact file name; `copyto` copies an individual file under that exact name instead of, like `copy`, into a directory |

</details>

Measure when the file appears in the mount. Repeat the test for modifications and deletion. The result depends on the backend, its polling support, and `--poll-interval` and `--dir-cache-time`. If the application must see current changes immediately, this behavior must explicitly be part of the acceptance criteria.

With the Remote Control interface enabled, you can specifically clear the directory cache:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `vfs/forget` | Remote Control command to run: clears the VFS's cached directory tree so the next access queries the remote again |

</details>

This is useful for a manual test, but it does not replace a suitable operational strategy.

## Put the cache under pressure

An almost empty cache is the simplest case. In a second test run, deliberately set `--vfs-cache-max-size` to a small value and read more data than fits in it.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `du -s` | Summarizes disk usage in one line instead of listing every subdirectory |
| `du -h` | Output in human-readable units (K, M, G) |
| `du --apparent-size` | Shows logical file size instead of actual disk space used |
| `find … -type f` | Considers only regular files, not directories |
| `wc -l` | Counts output lines, which here means the number of cache files |

</details>

The two sizes can differ greatly. In full mode, Rclone uses sparse files: a file shows its full logical size even though only the read portions consume local space.

The cache limit is also soft. Rclone checks it at the interval set by `--vfs-cache-poll-interval`, and open files cannot be removed. The cache may therefore temporarily exceed the limit. However, it should decrease again after files are closed and the next cleanup run occurs.

Record the peak value, the value after cleanup, and the time required. This lets you size the required local storage sensibly.

## Simulate two different failures

An unreachable cloud and a crashed Rclone process are two different failures:

| Failure | What it tests |
|---|---|
| Backend or network unreachable, Rclone continues running | Behavior during retries, timeouts, and with already cached files |
| Rclone process terminated | FUSE mount behavior and restoration of the mount point |

Simulate both only in the test environment. You can force-stop an Rclone container for the second case:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--signal KILL` | Sends SIGKILL instead of the default SIGTERM signal; the process has no opportunity to clean up |
| `<rclone-container>` | Name or ID of the Rclone container |

</details>

During the outage, check the application, not just the mount point:

- Which functions remain available?
- How long does an access wait before an error appears?
- Are fully cached files still accessible?
- Does the application stop new write operations?
- Does a clear error message appear, or only a hanging process?
- Does your monitoring alert?

When the mount is unavailable, a write service must not silently write into the underlying local directory. After the mount returns, those files would be hidden. A simple safeguard before every write job is:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-q` | No output; the result is available only through the exit code |
| `/mnt/rclone-test` | Path to check; exit code 0 only if a mount is actually active there |
| `\|\| exit 1` | Stops the script if the path is not a mount point |

</details>

After restarting Rclone, check the mount on the host and from every consuming container. A recreated mount reaches an already running container only with suitable mount propagation. For Docker, `rslave` is usually needed on the consuming side. Details are in the article [Operating Rclone Mounts Reliably in Docker](/blog/rclone-mount-in-docker-container).

## A concrete example from Paperless-ngx

For my Paperless test, I generated 40 PDFs totaling 13.9 MB. A previously unopened document took about 1.8 seconds, while an immediately repeated access took 19 to 24 milliseconds. A VFS cache limited to 4 MB briefly rose to 12.7 MiB and was cleaned up again during the next run.

While the remote was unreachable, the document list, full-text search, and thumbnails continued to work because that data was local. Only the original could not be opened. After rebuilding the mount, the running Paperless container could access the files again without being restarted itself.

These figures are not a benchmark for Rclone or Proton Drive. The behavior is what matters: hot storage remained available locally, cold reads were slower but predictable, and the service recovered after the failure.

## What belongs in the test report

A result that can be reviewed later includes at least:

- Rclone version and backend used
- operating system, FUSE variant, and file system of the cache directory
- complete mount command without credentials
- number, size distribution, and structure of the test files
- cold-read and warm-read values for multiple files
- write duration until visibility in the remote
- cache peak value and cleanup duration
- result of `rclone check --download`
- behavior during backend failure and when the Rclone process is terminated
- recovery time from the application's perspective
- retries, timeouts, throttling, and authentication errors from the log

Define a threshold for each item in advance. Then the test ends with a decision rather than just a collection of interesting figures.

## When the setup is ready

A cold-storage mount is ready for use if you can answer these questions with yes:

- Are cold reads fast enough for the intended service?
- Does the cache speed up repeated access as expected?
- Does local storage use remain manageable even under load?
- Do all files match after a complete download?
- Do all required file operations work with the selected backend?
- Does the application behave in a controlled manner during a cloud outage?
- Are write operations safely stopped when the mount is unavailable?
- Does a recreated mount reach all running consumers?
- Does monitoring report the outage before a user does?

If an answer is missing, you at least know exactly what you need to continue working on. That is far more helpful than a mount that looked good with the first `ls` and only shows its limitations in operation.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducible test files and directory structures with configurable sizes.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, writeback, sparse files, cache limits, and directory cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparison of source and destination, including full verification with `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): targeted clearing of the VFS directory cache with `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): complete reference for global options, including logging, statistics, and VFS parameters.
