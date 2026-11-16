---
title: "Pi-hole 2026.11.0: FTL 6.9 and Core 6.6 close eleven security vulnerabilities"
navTitle: "Pi-hole 2026.11.0"
description: "The Pi-hole 2026.11.0 Docker image bundles FTL v6.9 and Core v6.6 and fixes, among other things, admin code execution through files.gravity (CVSS 7.2) and several symlink vulnerabilities in the installer. Regex filters, webserver.acl, and the Sessions API behave differently in some respects; review these points after updating."
date: "2026-11-12"
kategorie: "Network and DNS"
timeToRead: "7 min read"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-and-core-6-6-close-eleven-security-vulnerabilities"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/en/blog/pi-hole-2026-11-0-ftl-6-9-and-core-6-6-close-eleven-security-vulnerabilities
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:41:33.627Z
translationReview: required
---

# Pi-hole 2026.11.0: FTL 6.9 and Core 6.6 close eleven security vulnerabilities

Pi-hole released the 2026.11.0 Docker image on November 11, 2026. It includes Pi-hole FTL v6.9 and Pi-hole Core v6.6, which were released the same day; according to the vendor, the Web Interface remains at v6.6.1. The two component releases fix eleven Security Advisories, including four rated High severity (CVSS 7.2 to 7.3). FTL v6.9 also updates dnsmasq, SQLite, and the TRE regex engine, and changes the behavior of regex filters, `webserver.acl` and parts of the API. The release is intended for all Pi-hole operators, both in Docker containers and as bare-metal installations. According to the release notes, its predecessor, 2026.10.1 from October 31, 2026, primarily introduced an updated Alpine base image (3.24.3).

## Version overview

| Version | Date | Type | Key points |
|---|---|---|---|
| Docker 2026.11.0 | 11/11/2026 | Image release | bundles FTL v6.9, Core v6.6, and Web v6.6.1; `setpriv` instead of `sudo`, logrotate through the container's crontab |
| FTL v6.9 | 11/11/2026 | Minor release | seven advisories (one High), dnsmasq v2.93+42, SQLite 3.54.1, TRE update with changed regex behavior |
| Core v6.6 | 11/11/2026 | Minor release | five advisories (four High, one Medium), installer writes to `/etc/pihole` only as user `pihole` |
| Web v6.6.1 | 10/17/2026 | unchanged | no new version in this release |

## High-severity security fixes

The advisories have been published since November 11, 2026. No CVE ID had been assigned to the reviewed advisories at that time; according to the text, one has been requested for the advisory concerning `files.gravity` (as of November 12, 2026).

- **GHSA-pj9m-9623-2g75 (FTL) and GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2:** Code execution by an authenticated admin (CWE-78, CWE-89). FTL 6.8 had relaxed path validation, allowing `files.gravity` to contain a single quotation mark. `gravity.sh` inserted this value unescaped into an SQL `ATTACH` statement; this allowed operating system commands to be executed as user `pihole` through FTL's sqlite3 shell, according to the advisory in both Docker and bare-metal installations. Core through 6.5 and FTL 6.8 are affected. Core now escapes the value, while FTL binds the path in Teleporter as a parameter. The vendor emphasizes that both packages must be updated together.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3:** Incomplete fix for the symlink vulnerabilities from October (CWE-59, CWE-61). `pihole -r` and `pihole -up` continued to follow symlinks in `/etc/pihole` when writing `install.log`, `migration_backup_v6`, and `hosts/custom.list`. This requires code execution as user `pihole` and an administrator who subsequently runs repair or an update as root. Versions through 6.5 are affected.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3:** root followed symlinks and paths from the configuration when downloading `macvendor.db`, during `pihole checkout`, and during `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3:** `pihole -d` followed symlinks in directories owned by user `pihole`. This allowed root to copy arbitrary files into the debug log and overwrite others or change their ownership.

Core v6.6 therefore writes `install.log`, the migration backup, `hosts/custom.list`, and `macvendor.db` only as user `pihole`. `pihole -d` also no longer operates as root in these directories.

## Password hashes in the debug log

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3:** The debug log from `pihole -d`, and therefore uploads to the Tricorder service, contained the web password hash, the app password hash, and the TOTP secret. Core v6.6 removes these values from the log. If you uploaded a debug log in the past, the vendor recommends changing the password and app passwords and setting up two-factor authentication again.

## Low-severity security fixes in FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8:** App sessions without `webserver.api.app_sudo` and CLI sessions could list all API sessions and terminate any of them.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7:** IPv4 entries in `webserver.acl` did not apply to IPv4 clients on a dual-stack `webserver.port` such as `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1:** A password change through `pihole-FTL --config` or `pihole setpassword` could be reversed by a simultaneous API configuration write; the old password and its sessions then remained valid.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2:** Queries recorded with `misc.privacylevel` 3 could enter the long-term database after lowering the level.
- **GHSA-9mh9-95wj-4w4q and GHSA-9hh4-j8jf-6c96, CVSS 2.2 each:** A crash while reading metrics during a hosts file reload and a use-after-free during a v5-format Teleporter import. Both require a valid session.

## Changed behavior after the update

According to the release notes, FTL v6.9 is deliberately not v6.8.1 because the TRE update changes how some stored regex filters are evaluated:

- **Regex:** A missing lower bound (`{,n}`, `{,}`) now means `{0,n}` or `{0,}`. More than three nested groups with approximate matching and back-references in unbounded repetitions are rejected.
- **Local DNS:** Entries in `dns.cnameRecords` with an invalid domain name or a TTL that does not fit in an integer are rejected.
- **webserver.acl:** IPv4 entries apply on all ports, including dual-stack ports. IPv6 entries other than `::ffff:a.b.c.d` now match only IPv6 clients; `[::]/0` therefore no longer permits IPv4 clients.
- **Sessions API:** App sessions without `webserver.api.app_sudo` and CLI sessions receive status 403 for `GET /api/auth/sessions` and `DELETE /api/auth/session/{id}`. A session can still terminate itself with `DELETE /api/auth`.
- **Web server:** Requests with more than 64 headers are rejected. `/api/info/system` returns an error instead of null values when the load averages cannot be read.
- **DHCP API:** `DELETE /api/dhcp/leases/{ip}` responds with 503 instead of 400 if the resolver has not processed the request in time.
- **Password:** A password set through `pihole setpassword` or `pihole-FTL --config` is no longer overwritten by the next configuration change in the Web Interface.

After updating, review regex filters with lower bounds or back-references, entries in `webserver.acl` (especially `[::]/0`), and scripts or integrations that list or terminate sessions using app passwords.

## DNS, DHCP, and other fixes in FTL v6.9

The embedded dnsmasq is now at v2.93+42 and additionally includes a fix in `blockdata_alloc()` that has not yet been accepted upstream. According to the release notes, fixes include a heap out-of-bounds read in `zone_status()` (DNSSEC), a heap out-of-bounds write in `blockdata_read()`, a DNS name buffer overflow, an off-by-one in the DHCPv6 FQDN option, as well as SERVFAIL with lower wildcard responses and lost responses during TCP pipelining. With DNSSEC enabled, dnsmasq reserves additional TCP slots for its own validation queries so validation also works when clients exhaust the TCP connection limit. Quad9's unfiltered upstream is now marked as DNSSEC-validating.

For configuration and the API, FTL v6.9 fixes, among other things, lost changes from two simultaneous configuration changes, a TCP worker that held `pihole.toml.lock` for up to 300 seconds, and ignored `FTLCONF_` variables when FTL started without `pihole.toml`. The web server could previously remain stuck at 100% CPU usage and stop accepting connections. In containers, `/api/info/system` previously reported the host's load. Also relevant for operations, `gravity.db` is now reopened only when the file has changed; according to the vendor, this accounted for the largest share of FTL's CPU load while idle. SQLite is now at 3.54.1.

## Changes in Core v6.6 and the Docker image

When dropping privileges from root to user `pihole`, Core v6.6 switches from `sudo` to `setpriv`; `sudo` is therefore no longer a dependency. According to the vendor, administrative commands such as `pihole -up` still require root privileges. Log rotation is now handled by the system's daily logrotate; Pi-hole's forced nightly rotation is removed because it conflicted with the system run and rotated logs twice. Further fixes concern `pihole flush quiet`, `pihole updatechecker`, and a warning when a local git repository is not marked as safe (previously, `pihole -v` then only displayed N/A). The unneeded `CAP_IPC_LOCK` capability has been removed.

The Docker image adopts the switch to `setpriv` and schedules log rotation through the container's crontab. The nightly image is now built from the `development` branch.

## Recommended approach

The target version is Docker image 2026.11.0 or, for bare-metal installations, FTL v6.9 with Core v6.6. Because the vulnerability through `files.gravity` is closed only with both components, update FTL and Core together. Beforehand, create a backup with Teleporter (Settings, Teleporter in the Web Interface, or `pihole-FTL --teleporter`), as recommended by the vendor, and read the changelogs because of the behavior changes. Update bare-metal installations with `pihole -up`. According to the documentation, `pihole -up` and `pihole -r` are disabled in containers; replace the image there. With Docker Compose, run the following commands in the directory containing `docker-compose.yml`, according to the documentation:

```bash
docker compose pull
docker compose down
docker compose up -d
```

For installations using `docker run`, the documentation specifies `docker pull pihole/pihole:latest`, followed by `docker stop pihole` and `docker rm pihole`, and a new `docker run` with the previous arguments. If you previously uploaded a debug log, also change the passwords and set up 2FA again. The release notes contain no information about known issues or support periods (as of November 12, 2026).

## Sources

1.  [GitHub: pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0): Release notes for the Docker image with component versions, Docker changes, and a list of the eleven advisories.

2.  [GitHub: pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9): Security fixes, dnsmasq and SQLite update, behavior changes for regex, ACL, and API.

3.  [GitHub: pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6): Core security fixes, switch to setpriv, changes to log rotation.

4.  [GitHub: pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1): Status of the Web Interface as of October 17, 2026, unchanged in 2026.11.0.

5.  [GitHub: pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1): Previous image with Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75): Admin code execution through files.gravity, CVSS 7.2, affected and fixed versions, CVE requested.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82): Incomplete symlink fix in the installer, CVSS 7.3, prerequisites.

8.  [Pi-hole Blog: Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/): Announcement with update guidance and recommendations after debug log uploads.

9.  [Pi-hole Docs: Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/): Update commands for Docker Compose and docker run; pihole -up is disabled in containers.
