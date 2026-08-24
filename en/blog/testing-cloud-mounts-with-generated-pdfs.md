---
title: "Testing cold storage with Rclone: a practical test plan"
navTitle: "Testing Rclone"
description: "Before a service reads its files from the cloud via an Rclone mount, you should check more than just directory access. This test plan covers cold reads, warm reads, write operations, cache behaviour, file integrity and failures."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 mins reading time"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
translationOf: "cloud-mount-testen-dummy-pdfs"
slug: "testing-cloud-mounts-with-generated-pdfs"
url: "https://rafaelpfister.ch/en/blog/testing-cloud-mounts-with-generated-pdfs"
translationId: article-8592f808b2e93cd4
translatedAt: 2026-07-28T12:20:26.034Z
translationReview: automatic
translationSourceHash: 4dd3058563b8e3853528cbd3cb5b216cc840923ceee9250055c3000c296232b9
---

A Rclone mount is quick to set up. The remote appears as a directory, `ls` displays files, and the initial functional test is passed. However, this says little about its suitability for production use.

As soon as a service accesses the mount, further questions arise: How long does it take to access a file for the first time? Which access requests are handled by the local cache? What happens to a file that hasn’t yet been uploaded if Rclone crashes? Will a running container still recognise the newly rebuilt mount? And how does the service react if the cloud is temporarily unavailable?

This article provides a general test plan for this. You can use it for a document archive, a media server, a photo management system or any other service that retrieves rarely used files from cold storage via Rclone.

## First, define what you want to achieve

Cold storage does not automatically mean the same thing for every application. A media server usually reads large files sequentially. A photo management system loads many small preview files and jumps to different locations. A document archive opens comparatively small files, but often only once.

Before testing, make a note of the key characteristics of your actual data set:

- typical file size and the largest file present
- number of files per directory
- full reads or random access to individual sections
- ratio of read to write accesses
- number of concurrent users or processes
- changes occurring directly on the remote system outside the mount
- acceptable wait time for a cold read
- maximum available space for the local cache

Only then can meaningful success criteria be established. Opening a single file in 1.2 seconds may be perfectly acceptable for an archive but unusable for an interactive application.

## Generating a reproducible test dataset

Rclone already provides a suitable tool for this. `rclone test makefiles` generates the same file tree every time using a fixed seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Adjust the number and sizes to match your actual dataset. Don’t just test average files. Some very small files demonstrate how costly metadata accesses are; some large files reveal throughput, read-ahead and cache behaviour.

Also include filenames that might cause problems in practice:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

The final test is particularly important if the local file system and cloud backend treat upper and lower case differently.

If your service only accepts specific formats, arbitrary binary files will not suffice. In that case, you should also generate synthetic files in precisely those formats. In the case of Paperless-ngx, these were PDFs with a genuine text layer, so that the test does not inadvertently measure OCR performance instead of the storage path. For a photo management system, the dataset should include different image sizes and formats; for a media server, short files with various codecs.

## A baseline measurement without mounting

Before FUSE and the VFS cache come into play, you should measure the backend directly. Copy the data set to the test remote using Rclone and save a detailed log:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Then check whether the source and destination match:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

With `--download` , Rclone actually reads the data and compares it, even if the backend does not provide matching hashes. This takes longer, but provides a useful starting point for the subsequent integrity test.

Record the upload time, transfer rate, number of retries and any API errors. If direct access is already unstable, mounting the volume will not resolve the issue.

## Separate the test mount from the production cache

Use a separate mount point and cache directory for the measurement:

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

These values are provided as an example and do not constitute a general recommendation. The key point is the separation: an empty test cache makes cold reads reproducible without you having to delete files from a running production cache.

`--vfs-cache-mode full` is usually the most informative test mode for applications. Rclone buffers read and write accesses locally and can better simulate file accesses that would not be possible with pure object storage. This additional compatibility comes at the cost of local storage space.

## Always test from the perspective of the actual service

A mount may work for your user but still be unusable for the service. Common causes include a different user ID, the absence of `--allow-other`, container boundaries, or incorrect mount propagation.

You should therefore carry out at least one complete read operation using the same identity under which the application will later run:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

If the service is running in Docker, the test should be carried out within the container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

Even better is a real-world application test. Open the file via the service’s web interface or API. This is the only way to detect whether the application, for example, initiates multiple parallel reads, jumps to the end of the file, or expects additional metadata.

## Measuring cold reads and warm reads separately

With `--vfs-cache-mode full` there are three layers between the application and the cloud:

| Layer | What is stored there |
|---|---|
| Remote | the complete file in the cloud service |
| VFS cache | locally stored sections of files that have already been read |
| Linux page cache | recently used data in RAM |

For a cold read, use a file whose contents have never been read via the test mount. For the warm read that follows immediately afterwards, the file is in the VFS cache and, in most cases, also in RAM.

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

Don’t just measure a single file. Use at least ten previously unread files of varying sizes and note down the median, the slowest value and the file size. A single best value is not a basis for decision-making.

A Warm Read is not a pure hard disk test, because the kernel can keep parts of the file in RAM. For most cold-storage scenarios, this is not a problem. What matters is what a user experiences when opening a file for the first time and on subsequent openings. If you want to evaluate RAM and the local disk separately, you must also check the page cache and clear it in a verifiable manner.

## Don’t just test complete read operations

`cat` reads a file from start to finish. Many applications behave differently:

- A video player first reads the header and index, then jumps to a different position and continues loading sequentially.
- An image manager reads metadata and then generates a thumbnail.
- An archiving programme may read the end of the file first.
- Multiple workers may access different files simultaneously.

Test these workflows with the actual application. Monitor the Rclone log and the cache in parallel. For large files, it is worth noting how much Rclone actually caches locally and whether `--vfs-read-ahead` matches the access pattern.

Furthermore, an Rclone mount is not a suitable storage location for databases or other files that require reliable locking and frequent changes within the same file. The VFS layer bridges the differences between the file system and object storage, but does not turn the backend into a local file system.

## Test the write path separately

If your service is read-only, mount the remote as read-only where possible. If it needs to write, test creating, overwriting, renaming and deleting files individually.

A written file does not necessarily appear immediately on the remote. When the VFS cache is active, the upload only begins after the file has been closed and `--vfs-write-back` has expired. You should therefore check both conditions:

1. The application has successfully closed the file.
2. The file is then readable on the remote via direct Rclone access.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Nach Ablauf von --vfs-write-back:
rclone cat remote:cold-storage-test/writeback-test.txt
```

Repeat the test with a large file and terminate Rclone whilst the upload is still in progress. Then restart using the same cache directory and check whether the upload continues. It is precisely this time window that determines how much data is at risk in the event of a server failure.

Also test renaming and deletion. Many cloud backends handle these operations differently to a local file system. It is not only relevant whether the command completes successfully, but also when the change becomes visible upon direct access to the remote system and to other clients.

## Testing changes outside the mount

Files can be modified via the provider’s web interface, a second Rclone process or another server. The mount does not always detect such changes immediately, as directory information is cached.

Therefore, use a second Rclone command to create a file directly on the remote:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

Note when the file appears in the mount. Repeat the test for modification and deletion. The result depends on the backend, its polling support, and `--poll-interval` and `--dir-cache-time` options. If the application needs to see recent changes immediately, this behaviour must be explicitly included in the acceptance criteria.

With the remote control interface enabled, you can selectively flush the directory cache:

```bash
rclone rc vfs/forget
```

This is useful for manual testing, but is no substitute for a suitable operational strategy.

## Putting the cache under pressure

An almost empty cache is the simplest case. In a second round of testing, deliberately set `--vfs-cache-max-size` to a small value and read more data than will fit into the cache.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

The two sizes may differ significantly. In full mode, Rclone uses sparse files: a file shows its full logical size, even though only the read sections occupy local space.

The cache limit is also soft. Rclone checks it at intervals set by `--vfs-cache-poll-interval`, and open files cannot be removed. The cache may therefore briefly exceed the limit. However, it should decrease again once the files have been closed and the next clean-up cycle has run.

Log the peak value, the value after the clean-up, and the time taken. This allows you to reasonably estimate the required local storage.

## Simulating two different failures

An unreachable cloud and a crashed Rclone process are two different errors:

| Failure | What you are testing |
|---|---|
| Backend or network unreachable, Rclone continues to run | Behaviour during retries, timeouts and with files already cached |
| Rclone process terminated | Behaviour of the FUSE mount and restoration of the mount point |

Simulate both scenarios only in the test environment. For the second scenario, you can force-terminate an Rclone container:

```bash
docker kill --signal KILL <rclone-container>
```

During the failure, test the application and not just the mount point:

- Which functions remain available?
- How long does an access attempt wait before an error is displayed?
- Are files that have already been fully cached still accessible?
- Does the application stop new write operations?
- Is a clear error message displayed, or does the process simply hang?
- Does your monitoring system trigger an alert?

A write service must not write unnoticed to the underlying local directory if the mount is missing. Once the mount returns, these files would be overwritten. A simple safeguard for every write job is:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

After restarting Rclone, check the mount on the host and from each consuming container. A newly established mount will only reach a container that is already running if mount propagation is configured correctly. For Docker, `rslave` is usually required on the consuming side. Full details can be found in the article [Running Rclone Mounts Reliably in Docker](/blog/rclone-mount-in-docker-container).

## A concrete example from Paperless-ngx

For my Paperless test, I generated 40 PDFs totalling 13.9 MB. A previously unopened document took around 1.8 seconds to load, whilst a repeat access took between 19 and 24 milliseconds. A VFS cache limited to 4 MB briefly rose to 12.7 MiB and was cleared again on the next run.

Whilst the remote was unreachable, the document list, full-text search and thumbnails continued to work because this data was stored locally. Only the original file could not be opened. Once the mount had been rebuilt, the running Paperless container was able to access the files again without needing to be restarted itself.

These figures are not a benchmark for Rclone or Proton Drive. What is interesting is the behaviour: hot storage remained available locally, cold reads were slower but predictable, and the service recovered after the outage.

## What should be included in the test log

A result that can be reproduced at a later date should include at least the following:

- Rclone version and backend used
- Operating system, FUSE variant and file system of the cache directory
- Full mount command without credentials
- Number, size distribution and structure of the test files
- Cold read and warm read values for several files
- Write time until visibility on the remote server
- Peak cache value and duration of the purge
- Result of `rclone check --download`
- Behaviour in the event of a backend failure and a terminated Rclone process
- Recovery time from the application’s perspective
- Retries, timeouts, throttling and authentication errors from the log

Define a threshold value for each point in advance. Then the test will conclude with a decision rather than just a collection of interesting figures.

## When the setup is ready

A cold storage mount is ready for use if you can answer ‘yes’ to the following questions:

- Are cold reads fast enough for the intended service?
- Does the cache accelerate repeated accesses as expected?
- Does the local storage requirement remain manageable even under load?
- Do all files match after a full download?
- Do all required file operations work with the chosen backend?
- Does the application behave in a controlled manner in the event of a cloud outage?
- Are write operations safely halted if the mount is missing?
- Does a newly established mount reach all active consumers?
- Does the monitoring system detect the failure before a user reports it?

If an answer is missing, at least you know exactly what you need to work on next. That’s far more helpful than a mount that looked fine on the first `ls` command but only reveals its limitations once it’s in production.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducible test files and directory structures with configurable sizes.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, writeback, sparse files, cache limits and directory cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): Comparison of source and destination, including a full check using `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): selectively flushing the VFS directory cache using `vfs/forget`.
