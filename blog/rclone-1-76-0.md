---
title: "Rclone 1.76.0: zahlreiche Sicherheitskorrekturen, VFS-Fixes und neue Backends"
navTitle: "Rclone 1.76.0"
description: "Rclone 1.76.0 vom 6. November 2026 schliesst Sicherheitslücken in serve, rc und mehreren Backends, darunter eine Auth-Proxy-Lücke in serve sftp mit CVSS 8.1, korrigiert fünf Fehler im VFS-Layer und blockiert HTTPS-zu-HTTP-Redirects standardmässig. Betreiber von Mounts, serve-Diensten und rc sollten aktualisieren und die geänderten Standards prüfen."
date: "2026-11-11"
kategorie: "Rclone"
timeToRead: "7 Min. Lesezeit"
themen:
  - "rclone"
produkte:
  - "rclone"
protokolle:
  - "releases"
  - "storage"
slug: "rclone-1-76-0"
translationId: article-aa7be1fdff77b8cc
url: https://rafaelpfister.ch/blog/rclone-1-76-0
---

# Rclone 1.76.0: zahlreiche Sicherheitskorrekturen, VFS-Fixes und neue Backends

Rclone 1.76.0 ist am 6. November 2026 erschienen, rund vier Wochen nach 1.75.2 (9. Oktober 2026). Der Changelog führt zwanzig Einträge im Abschnitt Security, darunter eine Lücke im Auth-Proxy von `rclone serve sftp` mit CVSS 8.1, Credential-Lecks bei Redirects in den Backends Swift, NetStorage, Seafile und Ulozto sowie mehrere Symlink-Ausbrüche im Backend `local`. Dazu kommen fünf Korrekturen im VFS-Layer, der `rclone mount` und die serve-Befehle trägt, Erweiterungen am S3-Backend, zwei neue Backends (Dosya und Infomaniak kDrive) und der neue Befehl `index`. Einige Standards ändern sich: HTTPS-zu-HTTP-Redirects werden abgelehnt, Multipath-TCP-Listener sind deaktiviert, und Glob-Muster in Filtern werden strenger ausgewertet. Das Release richtet sich an alle, die Rclone für Mounts, Synchronisation, Backups oder als Server mit `rclone serve` und `rclone rcd` betreiben. Hintergrund zu Mounts in Containern steht im Artikel [Rclone-Mounts in Docker zuverlässig betreiben](/blog/rclone-mount-in-docker-container).

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 1.76.0 | 06.11.2026 | Feature-Release | Sicherheitskorrekturen in serve sftp (CVSS 8.1), rc, lib/http, local und vier Backends; Go 1.27.3; fünf VFS-Fixes; Redirect-Schutz `--allow-https-downgrade`; neue Backends Dosya und Infomaniak kDrive; Befehl `index` |

Die bisher höchste Version war 1.75.2. Ein Patch-Release 1.76.x gab es bis zum 11. November 2026 nicht.

## Sicherheitskorrekturen mit Advisory

Die folgenden Lücken hat das Rclone-Projekt am 6. November 2026 als GitHub Security Advisory veröffentlicht. Alle nennen 1.76.0 als gepatchte Version, eine CVE-ID ist bei keinem vergeben (Stand 11. November 2026, im Changelog als „CVE-PENDING“ markiert).

- **GHSA-66qv-66mg-9h74, CVSS 8.1 (High):** Bei `rclone serve sftp` mit `--auth-proxy` enthielt der Cache-Schlüssel des Auth-Proxys die Art der Anmeldung nicht. Ein bekannter Public Key liess sich dadurch als Passwort wiederverwenden, wenn der Auth-Proxy Key- und Passwort-Anmeldung unterschiedlich behandelt, Angreifer und Opfer denselben Benutzernamen und dieselbe Client-IP nutzen und der Cache-Eintrag noch gültig ist (Standard 300 Sekunden Inaktivität). Gleiche Client-IPs entstehen etwa hinter NAT, einem Bastion-Host oder einem Load Balancer. Andere serve-Protokolle sind laut Advisory nicht betroffen. Das Feld zu den betroffenen Versionen nennt „>= 1.51.0, <= 1.76.0“, das Feld zur gepatchten Version 1.76.0; massgeblich ist der Fix-Eintrag im Changelog von 1.76.0.
- **GHSA-vmgx-g9c5-669p, CVSS 4.2:** Bei `serve sftp`, `serve ftp`, `serve http` und `serve webdav` mit `--auth-proxy` wirkte ein Widerruf im Auth-Proxy nicht, solange eine gecachte Antwort bestand. Ab 1.76.0 greift ein Widerruf innerhalb von fünf Minuten. `serve s3` ist nicht betroffen.
- **GHSA-hwj7-wp9g-wrv9, CVSS 5.3:** Über die Antwortzeit der bcrypt-Prüfung liessen sich bei `--htpasswd` und `--user` gültige von ungültigen Benutzernamen unterscheiden. Betroffen sind unter anderem `serve http`, `serve webdav`, `serve restic`, `serve s3`, `rcd` mit `--rc-htpasswd` und der Metrics-Endpunkt mit `--metrics-htpasswd`, Versionen 1.56.0 bis 1.75.2. Passwörter werden dabei nicht offengelegt.
- **GHSA-f8p4-8g4j-q3m2, CVSS 5.5:** Läuft der rc-Server ohne HTTP-Authentifizierung, konnte ein Client mit Zugriff auf den Port mit einem grossen JSON-Request an eine nicht autorisierte Methode den Speicher des Prozesses erschöpfen (Denial of Service, Versionen 1.46.0 bis 1.75.2). 1.76.0 begrenzt die Request-Grösse und lehnt nicht autorisierte Aufrufe vor dem Lesen des Bodys ab. Als Workaround nennt das Advisory HTTP-Authentifizierung für rc und ein Body-Limit am Reverse Proxy; die Bindung an localhost trennt laut Advisory lokale Benutzerkonten nicht voneinander.
- **GHSA-m97j-7x65-wh93, CVSS 4.8:** Die Backends Swift, NetStorage und Seafile gaben Zugangsdaten bei Redirects auf einen anderen Host oder von HTTPS auf HTTP weiter. Ausnutzbar ist das nur, wenn der konfigurierte Endpunkt selbst umleitet, etwa über einen Proxy. Als Workaround nennt das Advisory ausschliesslich `https://`-Endpunkte und keinen Proxy, der auf HTTP herabstuft.
- **GHSA-3xpx-hjg2-vrjh, CVSS 4.8:** Das Backend Ulozto sendete die Header `X-Auth-Token` und `X-User-Token` bei Redirects von Download-Links mit. Ein Angreifer braucht dafür eine Netzwerkposition, von der aus er den CDN-Verkehr umleiten kann.
- **GHSA-r5wv-vqqh-2r9c, CVSS 4.0:** Mit `--links` und `--metadata` konnte das Backend `local` über einen eingeschleusten Symlink `user.*`-Attribute ausserhalb des Zielverzeichnisses schreiben. Bis zum Update empfiehlt das Advisory, beide Optionen nicht gemeinsam zu verwenden, wenn Dritte das Ziel beeinflussen können.

Zusätzlich behebt 1.76.0 im Backend `local` weitere Fälle ohne eigenes Advisory, in denen `MkdirMetadata`, `Remove`, `Rmdir` und `DirMove` über Symlinks ausserhalb des Wurzelverzeichnisses wirken konnten.

## Abhängigkeiten und geänderte Standards

Rclone 1.76.0 ist mit Go 1.27.3 gebaut. Der Changelog nennt dazu CVE-2026-111943: Go ergänzt in `crypto/x509` eine Sperrliste für kompromittierte Zwischenzertifikate (CVSS 6.8 laut CISA-ADP-Bewertung, betroffen Go vor 1.26.10 und 1.27.0 bis vor 1.27.3). Ausserdem wurden `golang.org/x/net` auf v0.62.0 und `github.com/andybalholm/brotli` auf v1.2.7 aktualisiert, Letzteres wegen CVE-2026-111497; Hintergrund zu dieser Lücke steht im Artikel zu [CVE-2026-111497 im Brotli-Decoder](/blog/brotli-cve-2026-111497).

Multipath-TCP-Listener sind ab 1.76.0 standardmässig deaktiviert. Der Changelog begründet das mit CVE-2026-98432; der CVE-Eintrag beschreibt einen Use-after-free im TCP-Stack des Linux-Kernels (CVSS 9.8, korrigiert unter anderem in 6.1.190, 6.6.159, 6.12.113 und 6.18.56). Die Korrektur selbst liefern die Kernel-Updates der Distribution; Rclone verringert nur die eigene Angriffsfläche. Ein Flag zum Wiedereinschalten nennt der Changelog nicht.

Die Änderungen am Standardverhalten im Überblick:

- **Redirects:** Rclone lehnt Redirects von HTTPS auf HTTP ab. Das neue Flag `--allow-https-downgrade` lässt sie wieder zu.
- **Filter:** `[!...]` in Glob-Mustern ist nun eine negierte Zeichenklasse, `[^...]` schliesst `/` aus. Prüfen Sie Filterdateien und `--include`/`--exclude`-Regeln, die solche Klassen verwenden, vor dem produktiven Einsatz mit `--dry-run`.
- **local:** `--copy-links` folgt keinen Symlink-Schleifen mehr. Ohne Root-Rechte setzt Rclone nur noch die Besitzverhältnisse, die ein normaler Benutzer setzen darf. Fehlender Speicherplatz bricht Multi-Thread-Transfers jetzt mit einem fatalen Fehler ab.
- **Parameter-Prüfung:** Zu grosse Grössenangaben, NaN und überlaufende Dauern werden abgelehnt, ebenso `--bwlimit`-Zeitangaben ohne Doppelpunkt oder mit Vorzeichen.
- **bisync:** `--max-delete` berücksichtigt `--track-renames`; Werte über 100 werden abgelehnt.
- **operations:** `--immutable` wirkt nun auch bei `copyto`, `moveto` und beim Kopieren einzelner Dateien.
- **Deprecated:** `--rc-web-gui` sowie bei `serve s3` die Option `--no-cleanup`, die nun ein Alias des neuen `--directory-markers` ist.

Skripte und Units, die Rclone mit festen Parametern aufrufen, sollten Sie nach dem Update einmal ausführen und auf Fehlermeldungen zu abgelehnten Werten prüfen.

## Mount und VFS-Cache

Für `rclone mount` nennt der Changelog eine Korrektur: VFS-Instanzen wurden bei fehlgeschlagenen Mounts und beim Unmount nicht freigegeben. Im VFS-Layer, den Mount und die serve-Befehle gemeinsam nutzen, behebt 1.76.0 fünf Fehler:

- Ein verkleinerndes Truncate auf eine nicht gecachte Datei ging verloren.
- Das Umbenennen eines Verzeichnisses während des Ablaufs des Verzeichnis-Caches führte zum Absturz.
- Virtuelle Einträge blieben in gelöschten Verzeichnissen bestehen.
- Nach einer Umbenennung, die sich nur in Gross-/Kleinschreibung oder Unicode-Normalisierung unterschied, erschienen Einträge doppelt.
- Uploads einer geänderten Datei starteten nicht, solange die Datei weiter gelesen wurde.

Der letzte Punkt betrifft Mounts mit VFS-Cache, auf denen eine geänderte Datei weiter gelesen wird, während sie hochgeladen werden soll. Beobachten Sie in solchen Setups nach dem Update die Upload-Warteschlange und führen Sie einen Schreib- und Lesetest durch, wie im [Testplan für Rclone-Mounts](/blog/cloud-mount-testen-dummy-pdfs) beschrieben.

## S3-Backend und Proton Drive

Im S3-Backend puffert Rclone Uploads mit Object Lock nicht mehr zwischen, wenn der MD5-Wert der Quelle bekannt ist. Neu ist der Backend-Befehl `link` mit signierten Overrides für Response-Header. Hinzu kommen die Regionen Fastly EU Central 1 und Scaleway IT-MIL, mehrere Huawei-OBS-Regionen sowie die Anbieter Autonomys Auto Drive, Clever Cloud Cellar, Infomaniak Swiss Backup, Neon Object Storage und RelAix.Storage. Multi-Thread-Uploads berechnen Checksummen in Lesevorgängen von 1 MiB, nach einem Restore wird die tatsächliche Storage Class angezeigt. Korrigiert wurden `rmdir` auf der Root-Ebene mit `--s3-directory-markers` und presigned URLs für GCS, Linode und Ceph.

Für `rclone serve s3` bringt 1.76.0 das Flag `--directory-markers` sowie Änderungen bei Tags, ETags, Multipart-ETags und Conditional Writes. Behoben ist ausserdem, dass sich Buckets nach einem abgebrochenen Multipart-Upload nicht löschen liessen.

Zum Backend Proton Drive enthält der Changelog von 1.76.0 keinen Eintrag. Die letzten Korrekturen dazu stammen aus 1.75.0 (Retry-Logik, Hänger bei Validierungsfehlern, falsche Änderungszeiten) und 1.75.1 (mit 1.75.0 hochgeladene Dateien liessen sich in den Proton-Apps nicht lesen). Den Stand der Linux-Unterstützung von Proton Drive beschreibt der Artikel [Proton Drive unter Linux](/blog/proton-drive-linux-status).

## Weitere Neuerungen

Die beiden neuen Backends sind Dosya (Kaya) und Infomaniak kDrive. Der neue Befehl `index` schreibt statische Verzeichnislisten in ein Remote und kann bei Folgeläufen nur geänderte Verzeichnisse neu indexieren. Rclone unterstützt CRC-32C als Hash, `sync` kann die Reihenfolge der Transfers nach Mustern steuern, und die Verzeichnisansicht von `serve http` und `serve webdav` ist sortiert und neu gestaltet. Die GUI gilt laut Changelog nicht mehr als Beta. Die Statistik zählt `totalBytes` bei serverseitigen Kopien nicht mehr doppelt, und `serve sftp` meldet bei `rmdir` auf eine Datei einen Fehler, statt die Datei zu löschen.

## Empfohlenes Vorgehen

Zielversion ist 1.76.0. Besonders dringend ist das Update auf Systemen mit `rclone serve sftp --auth-proxy`, mit `rclone rcd` oder `--rc` ohne HTTP-Authentifizierung, mit `--htpasswd`-geschützten serve-Diensten sowie bei Remotes vom Typ Swift, NetStorage, Seafile oder Ulozto. Bei Installationen aus den offiziellen Binaries aktualisiert `rclone selfupdate` die ausführbare Datei und prüft den Download per Hash und Signatur; mit `rclone selfupdate --check` zeigt Rclone vorher an, welche Version installiert würde. Bei Installationen aus deb- oder rpm-Paketen nennt die Dokumentation `--package deb` bzw. `--package rpm`. Laufende Mounts und serve-Prozesse verwenden die neue Version erst nach einem Neustart. Prüfen Sie danach Filterregeln mit Zeichenklassen, `--bwlimit`-Zeitpläne und Setups, die auf HTTP-Redirects angewiesen sind. Known Issues zu 1.76.0 nennen Changelog und Release-Seite nicht (Stand 11. November 2026).

## Quellen

1.  [Rclone: Changelog](https://rclone.org/changelog/): Einträge zu 1.76.0 (6. November 2026) sowie 1.75.0 bis 1.75.2 mit Sicherheitskorrekturen, Mount-, VFS-, S3- und Proton-Drive-Änderungen.

2.  [GitHub: rclone v1.76.0](https://github.com/rclone/rclone/releases/tag/v1.76.0): Release-Seite mit Binaries und Verweis auf den Changelog.

3.  [GitHub: Security Advisories von rclone](https://github.com/rclone/rclone/security/advisories): Übersicht der am 6. November 2026 veröffentlichten Advisories.

4.  [GHSA-66qv-66mg-9h74](https://github.com/rclone/rclone/security/advisories/GHSA-66qv-66mg-9h74): serve sftp, Auth-Proxy-Cache ohne Art der Anmeldung, CVSS 8.1.

5.  [GHSA-vmgx-g9c5-669p](https://github.com/rclone/rclone/security/advisories/GHSA-vmgx-g9c5-669p): Widerruf im Auth-Proxy wirkt bei gecachter Antwort nicht, CVSS 4.2.

6.  [GHSA-hwj7-wp9g-wrv9](https://github.com/rclone/rclone/security/advisories/GHSA-hwj7-wp9g-wrv9): Benutzernamen-Erkennung über bcrypt-Timing bei `--htpasswd`, CVSS 5.3.

7.  [GHSA-f8p4-8g4j-q3m2](https://github.com/rclone/rclone/security/advisories/GHSA-f8p4-8g4j-q3m2): Speichererschöpfung im rc-Server durch übergrosse JSON-Requests, CVSS 5.5.

8.  [GHSA-m97j-7x65-wh93](https://github.com/rclone/rclone/security/advisories/GHSA-m97j-7x65-wh93): Credential-Lecks bei Redirects in Swift, NetStorage und Seafile, CVSS 4.8.

9.  [GHSA-3xpx-hjg2-vrjh](https://github.com/rclone/rclone/security/advisories/GHSA-3xpx-hjg2-vrjh): Token-Leck bei Redirects im Backend Ulozto, CVSS 4.8.

10.  [GHSA-r5wv-vqqh-2r9c](https://github.com/rclone/rclone/security/advisories/GHSA-r5wv-vqqh-2r9c): Symlink-Ausbruch bei `--links --metadata` im Backend local, CVSS 4.0.

11.  [CVE.org: CVE-2026-111943](https://www.cve.org/CVERecord?id=CVE-2026-111943): Sperrliste für kompromittierte Zwischenzertifikate in Go `crypto/x509`, behoben in Go 1.27.3.

12.  [CVE.org: CVE-2026-98432](https://www.cve.org/CVERecord?id=CVE-2026-98432): Use-after-free im TCP-Stack des Linux-Kernels, CVSS 9.8, Anlass für die deaktivierten Multipath-TCP-Listener.

13.  [Rclone: rclone selfupdate](https://rclone.org/commands/rclone_selfupdate/): Dokumentation zu Selbstaktualisierung, `--check` und `--package`.

14.  [CVE-2026-111497: Lücke im Brotli-Decoder](/blog/brotli-cve-2026-111497): Hintergrund zur Brotli-Lücke, die 1.76.0 über das Abhängigkeits-Update adressiert.
