---
title: "When the log fills the disk: properly limiting log4j2 RollingFile, using totemomail as an example"
navTitle: "log4j2 disk space"
description: "A log volume filling up can bring down the entire gateway in the worst case. Why combining time- and size-based rotation without %i creates a single huge file, how strategy.max caps retention, the role of the log level, and where totemomail hides these values."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 min read"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "when-the-log-fills-the-disk-properly-limiting-log4j2-rollingfile-using-totemomail-as-an-example"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/en/blog/when-the-log-fills-the-disk-properly-limiting-log4j2-rollingfile-using-totemomail-as-an-example
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:15:49.077Z
translationReview: automatic
---

# When the log fills the disk: properly limiting log4j2 RollingFile, using totemomail as an example

A Java-based mail gateway writes surprising amounts of data when running at DEBUG level. A single busy day can generate several gigabytes of trace logs, and if the log volume is undersized, it fills up. The consequences are unpleasant: the Java process can no longer write to its log, the logging framework enters an error state, and even after space is freed up, it will not resume writing until after a restart. On a mail gateway, a full disk can also disrupt spooling and delivery. The cause is almost always log rotation that is configured but does not work as assumed.

The following article explains log4j2 rotation at precisely this point, first in general and then specifically for totemomail (which is based on Apache James and log4j2). The core issue is a single, easily overlooked entry in the file pattern.

## How log4j2 rotates

The `RollingFileAppender` of log4j2 combines two building blocks: one or more **TriggeringPolicies** decide *when* rotation occurs, and a **RolloverStrategy** decides *how* the archive files are named and how many are retained. Two policies are typically used at the same time:

- `TimeBasedTriggeringPolicy`: rotates by time, usually daily.
- `SizeBasedTriggeringPolicy`: rotates as soon as the active file reaches a size, such as 100 MB.

During rollover, the active file is renamed and archived. The archive file name is determined by `filePattern`, and it contains two placeholders whose interaction makes all the difference.

<details class="options-details">
<summary>Options at a glance</summary>

| Placeholder | Meaning |
|---|---|
| `%d{...}` | Date/time of the rollover according to the specified pattern, e.g. `%d{yyyy-MM-dd}` for the day |
| `%i` | The calculated archive file index, a counter that increments with each rollover |
| `%03i` | The same index, zero-padded to three digits |
| `.gz` / `.zip` at the end of the pattern | The archive is compressed during rollover |

</details>

The full reference is available in the log4j2 documentation for the Rolling File Appender; the table above lists only the elements relevant to size- and time-based rotation.

## The %i trap

This is exactly where the disk-filling error lies. If you name files only by date, i.e., `filePattern = trace.log.%d{yyyy-MM-dd}`, while also setting a 100 MB size policy, you do not get many 100 MB files per day, but one single file that continues to grow unchecked. Size-based rotation has no separate destination to write the next chunk to because the pattern contains no counter. The log4j2 documentation is clear on this point:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Without `%i`, the combination of time- and size-based rotation is therefore faulty; depending on the strategy, the file is either overwritten or grows beyond the configured size. In practice, this means the 100 MB limit never takes effect, a busy day writes everything to one file, and that file grows to several gigabytes. The fix is to add to the pattern:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

This causes every 100 MB rollover to create a separate indexed file (`trace.log.2026-09-04.1`, `.2`, `.3`), and the size limit works as intended.

## Retention through strategy.max

The index is also a prerequisite for retention to work. The `DefaultRolloverStrategy` has an attribute called `max`, which specifies the maximum number of archive files to retain; beyond this limit, the oldest files are deleted. Without `%i`, there is no index for `max` to count, so nothing is deleted and old dated files accumulate.

<details class="options-details">
<summary>Options explained</summary>

| Attribute | Effect |
|---|---|
| `max` | Maximum number of archive files retained; the oldest are removed beyond this limit |
| `min` | Lowest index value (default 1) |
| `fileIndex="min"` | The newest file receives index `min`, the oldest `max` |
| `fileIndex="max"` (default) | The oldest file receives index `min`, the newest `max` |
| `fileIndex="nomax"` | Nothing is ever deleted; new archives receive continuously increasing indexes |

</details>

The overall limit results from size and quantity: 100 MB per file times `max=10` caps the log at around one gigabyte, regardless of how much is written. If you need finer control over age rather than count, add a `Delete` action with `IfLastModified` (age) or `IfAccumulatedFileSize` (total size) to the strategy; for most cases, the combination of size per file and `max` is sufficient.

## The log level as the actual volume driver

Rotation and retention limit disk usage, but they do not change how much is written in the first place. The biggest lever is the log level. A gateway running in production at DEBUG logs every processing step of every message, and under load that adds up to gigabytes per day. For normal operation, the level should be INFO or higher; DEBUG is a tool for limited analysis, not continuous operation. If the level is set to INFO and size-based rotation is also correctly configured with `%i`, the two work together: INFO keeps the daily volume low, and rotation caps even a DEBUG outlier.

## Where totemomail keeps these values

In totemomail, these settings are not found in a local `log4j2.xml`, which can easily lead troubleshooting in the wrong direction. The configuration is generated at runtime from properties with the prefix `totemo.log4j2.*`, and these properties are managed centrally through the Management Console (Logging + Tracking section). Searching the file system for `log4j2.xml` is therefore fruitless; a `log4j.xml` in the configuration directory belongs to a bundled component (openjms) and has nothing to do with the trace log.

The relevant properties and their meaning:

<details class="options-details">
<summary>Options explained</summary>

| Property | Meaning |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | The file pattern; this is where `%i` belongs |
| `totemo.log4j2.appender.a1.policies.size.size` | Size per file for the SizeBasedTriggeringPolicy, e.g. `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Number of retained archive files |
| `totemo.log4j2.rootLogger.level` | Level of the log4j2 root logger |
| `totemo.log.priority` | Application-wide logging priority, the actual DEBUG switch |
| `totemo.tracking` | Detail level of message tracking; `debug` generates lines for each Mailet |

</details>

The dual nature is important: the log4j2 loggers can be set to `warn` or `error` and still generate a flood of DEBUG entries in the trace log because `totemo.log.priority` and `totemo.tracking` act as separate, higher-level switches. To reduce volume, set `totemo.log.priority` to INFO and change `totemo.tracking` from `debug` to `on`; this removes the detailed processing lines. Because the values are managed through the Console, they apply cluster-wide, and some require an instance restart to take effect (this is noted on the respective property).

## Restarting after the disk fills up

One detail that is easy to miss: after the disk has filled up once, logging does not return on its own, even if you free up space. The file appender remains in its error state until the Java process restarts. You can tell because the gateway still accepts and processes mail (the SMTP banner displays the correct time), but the trace log stops at the time the disk filled up. A controlled restart of the instance restores logging and also activates changed appender settings such as the new `filePattern`.

## Diagnosis in a few commands

The full partition and its cause can be narrowed down quickly. First, identify which file system is affected:

```bash
df -h
```

If the log volume is at 100 percent, a size-sorted listing identifies the primary culprit:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

If you find one daily file several gigabytes in size instead of many small indexed archives, that is the `%i` trap. After the fix and a restart, the file listing confirms that rotation is working:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

You should expect `trace.log` plus indexed archives `trace.log.<datum>.1`, `.2`, and so on, each approximately at the configured maximum size.

## Summary

Anyone using log4j2 with time- and size-based rotation absolutely needs `%i` in `filePattern`; otherwise, a single file grows unchecked and the size limit remains ineffective. Through `strategy.max` (together with the index), the number of archives caps disk usage, while the log level determines the volume at the source. In totemomail, these values are found in the Management Console under `totemo.log4j2.*` as well as in the higher-level switches `totemo.log.priority` and `totemo.tracking`; after the disk fills up, restarting the instance is required for logging to write again.

## Sources

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): Reference for filePattern, TriggeringPolicies, and DefaultRolloverStrategy, including the statement on `%i` for time-based rotation.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): Context on appenders, layouts, and logger hierarchy, for understanding the root logger and log level.
