---
title: "Testing Cold Storage with Rclone: a practical test plan"
navTitle: "Testing Rclone"
description: "Before a service reads its files through an Rclone mount from the cloud, you should check more than directory access. This test plan covers Cold Reads, Warm Reads, writes, cache behavior, file integrity, and outages."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min to read"
themen:
  - "rclone"
related:
  - "rclone-mount-inside-docker-container"
  - "offloading-paperless-documents-to-cloud-storage"
translationOf: "cloud-mount-testen-dummy-pdfs"
slug: "testing-cloud-mounts-with-generated-pdfs"
url: "https://rafaelpfister.ch/en/blog/testing-cloud-mounts-with-generated-pdfs"
---

An Rclone mount is quick to set up. The remote shows up as a directory, `ls` lists files, and the first functional test passes. That says very little about production readiness.

As soon as a service starts accessing the mount, more questions come up: how long does the first access to a file take? Which accesses does the local cache serve? What happens to a file that hasn't been uploaded yet if Rclone crashes? Does a running container see the mount again once it's rebuilt? And how does the service react when the cloud is temporarily unreachable?

This article provides a general test plan for exactly that. You can use it for a document archive, a media server, a photo library, or any other service that fetches rarely needed files from a Cold Storage through Rclone.

## First, decide what you're trying to achieve

Cold Storage doesn't automatically mean the same thing for every application. A media server mostly reads large files sequentially. A photo library loads lots of small preview files and jumps around between different positions. A document archive opens comparatively small files, but often only once.

Before testing, note down the key characteristics of your real data set:

- typical file size and the largest file that occurs
- number of files per directory
- full sequential reads or random access to specific ranges
- ratio between read and write access
- number of concurrent users or processes
- changes that happen directly on the remote, outside the mount
- acceptable wait time for a Cold Read
- maximum space available for the local cache

Only from that do meaningful success criteria emerge. Opening a single file in 1.2 seconds can be perfectly fine for an archive and unusable for an interactive application.

## Generating a reproducible test data set

Rclone already ships a suitable tool for this. `rclone test makefiles` generates the same file tree every time, given a fixed seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Adjust the count and sizes to your real data set. Don't test only average-sized files. Some very small files show how expensive metadata access is; some large files reveal throughput, read-ahead and cache behavior.

Also add file names that tend to cause trouble in practice:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

The last test matters most when the local file system and the cloud backend handle case sensitivity differently.

If your service only accepts specific formats, arbitrary binary files aren't enough. In that case, also generate synthetic files in exactly those formats. For Paperless-ngx, that meant PDFs with a real text layer, so the test wouldn't accidentally measure OCR performance instead of the storage path. For a photo library, the data set should include different image sizes and formats; for a media server, short files with different codecs.

## A baseline measurement without a mount

Before FUSE and the VFS cache come into play, you should measure the backend directly. Copy the data set to the test remote with Rclone and keep a detailed log:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Then check whether source and destination match:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

With `--download`, Rclone actually reads the data and compares it, even if the backend doesn't provide matching hashes. That takes longer but gives you a solid baseline for the later integrity test.

Record upload time, transfer rate, number of retries, and API errors. If direct access is already unstable, the mount can't fix that.

## Keeping the test mount separate from the production cache

Use a dedicated mount point and a dedicated cache directory for the measurement:

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

These values are an example, not a general recommendation. What matters is the separation: an empty test cache makes Cold Reads reproducible without you having to delete files from a running production cache.

`--vfs-cache-mode full` is usually the most revealing test mode for applications. Rclone buffers reads and writes locally and can emulate file access patterns that wouldn't be possible on a plain object store. That extra compatibility costs local disk space.

## Always test from the real service's point of view

A mount can work fine for your user account and still be unusable for the service. Common causes are a different user ID, a missing `--allow-other`, container boundaries, or the wrong mount propagation.

So run at least one full read access under the same identity the application will later run as:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

If the service runs in Docker, the test belongs inside the container:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

Even better is a real application-level test. Open the file through the service's web interface or API. That's the only way you'll notice, for example, whether the application starts several parallel reads, seeks to the end of the file, or expects additional metadata.

## Measuring Cold Reads and Warm Reads separately

With `--vfs-cache-mode full`, three layers sit between the application and the cloud:

| Layer | What lives there |
|---|---|
| Remote | the complete file at the cloud service |
| VFS cache | locally stored ranges of already-read files |
| Linux page cache | recently used data in RAM |

For a Cold Read, pick a file whose contents have never been read through the test mount. On the immediately following Warm Read, it sits in the VFS cache and, most of the time, in RAM as well.

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

Don't measure just one file. Use at least ten previously unread files of different sizes and record the median, the slowest value, and the file size. A single best-case number is not a basis for a decision.

A Warm Read is not a pure disk test, because the kernel can keep parts of the file in RAM. For most Cold Storage scenarios that's not a problem. What matters is what a user experiences on first and repeated opens. If you want to judge RAM and local disk separately, you additionally need to control the page cache and demonstrably clear it.

## Don't test only full sequential reads

`cat` reads a file from beginning to end. Many applications behave differently:

- A video player first reads the header and index, later seeks to a different position, and then loads sequentially from there.
- A photo manager reads metadata and then generates a thumbnail.
- An archiving tool may read the end of the file first.
- Multiple workers may access different files concurrently.

Test these patterns with the actual application. Watch the Rclone log and the cache in parallel. For large files, it's interesting to see how much Rclone actually stores locally and whether `--vfs-read-ahead` matches the access pattern.

An Rclone mount is also not a sensible storage location for databases or other files that need reliable locking and frequent in-place changes. The VFS layer smooths over the differences between a file system and an object store, but it doesn't turn the backend into a local file system.

## Sign off on the write path separately

If your service only reads, mount the remote read-only if at all possible. If it has to write, test create, overwrite, rename and delete individually.

A written file doesn't necessarily show up on the remote right away. With an active VFS cache, the upload only starts after the file has been closed and `--vfs-write-back` has elapsed. So check both states:

1. The application has successfully closed the file.
2. The file is then readable on the remote through a direct Rclone access.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# After --vfs-write-back has elapsed:
rclone cat remote:cold-storage-test/writeback-test.txt
```

Repeat the test with a large file and kill Rclone while the upload is still running. Then restart it with the same cache directory and check whether the upload resumes. That exact window determines how much data is at risk if the server fails.

Also test renaming and deleting. Many cloud backends implement these operations differently than a local file system. What matters isn't just whether the command completes successfully, but when the change becomes visible on a direct access to the remote and to other clients.

## Testing changes made outside the mount

Files can be changed through the provider's web interface, a second Rclone process, or another server. The mount doesn't always see such changes immediately, because directory information is cached.

So create a file directly on the remote with a second Rclone invocation:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

Measure when the file shows up in the mount. Repeat the test for changes and deletions. The result depends on the backend, its polling support, and `--poll-interval` and `--dir-cache-time`. If the application needs to see current changes immediately, this behavior belongs explicitly in your acceptance criteria.

With the remote control interface enabled, you can deliberately drop the directory cache:

```bash
rclone rc vfs/forget
```

That's useful for a manual test but doesn't replace a proper operational strategy.

## Putting the cache under pressure

A nearly empty cache is the easy case. In a second test round, deliberately set `--vfs-cache-max-size` small and read more data than fits.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

The two figures can differ substantially. In full mode, Rclone uses sparse files: a file shows its full logical size even though only the ranges actually read occupy local space.

The cache limit is also soft. Rclone checks it at the interval set by `--vfs-cache-poll-interval`, and open files can't be evicted. So the cache is allowed to exceed the limit briefly. It should shrink back down, though, once the files are closed and the next cleanup pass runs.

Log the peak value, the value after cleanup, and the time cleanup took. That lets you size the required local storage sensibly.

## Simulating two different kinds of failure

An unreachable cloud and a crashed Rclone process are two different failures:

| Failure | What it tests |
|---|---|
| Backend or network unreachable, Rclone keeps running | Behavior under retries, timeouts and already-cached files |
| Rclone process terminated | Behavior of the FUSE mount and recovery of the mount point |

Simulate both only in the test environment. For the second case, you can hard-kill an Rclone container:

```bash
docker kill --signal KILL <rclone-container>
```

During the outage, check the application, not just the mount point:

- Which functions remain available?
- How long does an access wait before an error appears?
- Are already fully cached files still reachable?
- Does the application stop new writes?
- Does it produce an understandable error message, or just a hanging process?
- Does your monitoring fire?

A write-capable service must not silently write into the underlying local directory when the mount is missing. Once the mount returns, those files would be shadowed. A simple guard to put in front of every write job is:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

After restarting Rclone, check the mount on the host and from every consuming container. A freshly rebuilt mount only reaches an already-running container with the right mount propagation. For Docker, that usually means `rslave` on the consuming side. The details are in the article [Running Rclone mounts reliably in Docker](/en/blog/rclone-mount-inside-docker-container).

## A concrete example from Paperless-ngx

For my Paperless test, I generated 40 PDFs totaling 13.9 MB. A previously unopened document took around 1.8 seconds; a directly repeated access took 19 to 24 milliseconds. A VFS cache capped at 4 MB briefly rose to 12.7 MiB and was cleaned back up on the next pass.

While the remote was unreachable, the document list, full-text search and thumbnails kept working, because that data lived locally. Only the original file failed to open. After the mount was rebuilt, the running Paperless container could access the files again without being restarted itself.

These numbers aren't a benchmark for Rclone or Proton Drive. What's interesting is the behavior: Hot Storage stayed available locally, Cold Reads were slower but predictable, and the service recovered after the outage.

## What belongs in the test report

A result that's traceable later includes at least:

- Rclone version and backend used
- operating system, FUSE variant, and file system of the cache directory
- the full mount command, without credentials
- number, size distribution and structure of the test files
- Cold Read and Warm Read values for several files
- write duration until visible on the remote
- cache peak value and cleanup duration
- result of `rclone check --download`
- behavior on backend outage and on a terminated Rclone process
- recovery time from the application's point of view
- retries, timeouts, throttling and authentication errors from the log

Define a threshold for each point in advance. Then the test ends with a decision, not just a collection of interesting numbers.

## When the setup is ready

A Cold Storage mount is ready for production when you can answer yes to these questions:

- Are Cold Reads fast enough for the intended service?
- Does the cache speed up repeated access as expected?
- Does local storage demand stay under control even under load?
- Do all files check out after a full download?
- Do all required file operations work with the chosen backend?
- Does the application behave in a controlled way during a cloud outage?
- Are writes safely stopped when the mount is missing?
- Does a freshly rebuilt mount reach every running consumer?
- Does your monitoring show the outage before a user reports it?

If an answer is missing, at least you know exactly what to work on next. That's far more useful than a mount that looked good on the first `ls` and only showed its limits in production.

## Sources

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducible test files and directory structures with configurable sizes.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS cache modes, writeback, sparse files, cache limits, and the directory cache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): comparing source and destination, including a full check with `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): selectively dropping the VFS directory cache with `vfs/forget`.
