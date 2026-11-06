---
title: "Kiteworks EPG 9.5.1: Vier öffentlich dokumentierte Sicherheitskorrekturen"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 behebt vier öffentlich dokumentierte Schwachstellen, darunter eine Kontoübernahme mit CVSS 9.4. Installationen vor 9.5.1 sollten auf diese oder eine spätere Version aktualisiert werden."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 Min. Lesezeit"
themen:
  - "totemomail"
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-releases-9-5-1"
translationId: article-d4b9d984c72a54ea
url: https://rafaelpfister.ch/blog/kiteworks-epg-releases-9-5-1
---

# Kiteworks EPG 9.5.1: Vier öffentlich dokumentierte Sicherheitskorrekturen

Kiteworks führt die frühere totemo-Technologie als Email Protection Gateway (EPG) weiter. Version 9.5.1 war spätestens am 25. September 2026 die aktuelle Kiteworks-Version; ein öffentliches Veröffentlichungsdatum nennt der Hersteller nicht. Die am 30. September veröffentlichten Hersteller-Advisories dokumentieren für EPG 9.5.1 vier Korrekturen: eine Kontoübernahme, zwei nicht authentifizierte Fehler zum Schreiben von Dateien beziehungsweise zum Auslösen eines Denial of Service sowie einen Fehler, der Administratoren das Schreiben beliebiger Dateien erlaubte. Die Einordnung für die gleichnamige totemomail-Version steht im Artikel zu [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 9.5.1 | spätestens 25.09.2026 öffentlich bestätigt | Sicherheitsupdate | Vier öffentlich dokumentierte EPG-Korrekturen, darunter CVE-2026-102149 mit CVSS 9.4 |

## Sicherheitskorrekturen in 9.5.1

Alle vier Advisories betreffen laut Kiteworks sämtliche EPG-Versionen vor 9.5.1 und nennen 9.5.1 als korrigierte Version. Die Advisories wurden am 30. September 2026 veröffentlicht. Sie beschreiben nachträglich den Sicherheitsstand des Releases, nicht dessen genaues Veröffentlichungsdatum.

- **CVE-2026-102149, CVSS 9.4 (kritisch):** Eine Schwäche in der Zugriffskontrolle konnte einem Angreifer über das Netzwerk Zugang zu Benutzerkonten ermöglichen. Die CVSS-Bewertung setzt weder Berechtigungen noch Benutzerinteraktion voraus. Kiteworks ordnet die Ursache CWE-306 (fehlende Authentifizierung für kritische Funktionen) und CWE-639 (Autorisierungsumgehung über einen vom Benutzer kontrollierten Schlüssel) zu.
- **CVE-2026-102143, CVSS 7.5 (hoch):** Ein nicht authentifizierter Angreifer konnte Dateien auf die Appliance schreiben. Kiteworks nennt ein Authentifizierungsproblem sowie CWE-306 und CWE-434 (unbeschränkter Upload gefährlicher Dateitypen).
- **CVE-2026-102146, CVSS 6.5 (mittel):** Ein Administrator konnte beliebige Dateien schreiben. Kiteworks beschreibt einen Berechtigungsfehler; die CVSS-Metrik verlangt hohe Berechtigungen. Als Schwächen nennt der Hersteller CWE-73 (externe Steuerung von Dateiname oder Pfad) und CWE-1336 (unzureichende Neutralisierung in einer Template Engine).
- **CVE-2026-102144, CVSS 5.3 (mittel):** Ein Angreifer im Netzwerk konnte einen Denial of Service auslösen. Das Advisory nennt fehlende Authentifizierung für kritische Funktionen und unkontrollierten Ressourcenverbrauch (CWE-306, CWE-400). Auch diese Bewertung setzt keine Berechtigungen und keine Benutzerinteraktion voraus.

Für Betreiber ist die Kontoübernahme die vorrangige Korrektur, weil sie laut Advisory ohne vorhandenes Konto möglich ist. Die beiden weiteren Fehler ohne Anmeldung betreffen ebenfalls alle Versionen vor 9.5.1. Die Hersteller-Advisories veröffentlichen keine technischen Details zu Endpunkten, Konfigurationen oder Workarounds.

## Empfohlenes Vorgehen

Setzen Sie EPG mindestens auf 9.5.1, sofern eine ältere Version eingesetzt wird. Kiteworks nennt für alle vier Advisories ausdrücklich das Upgrade auf 9.5.1 oder später als Abhilfe. Ist bereits eine spätere Hersteller-Version freigegeben, erfüllt sie diese Mindestversion ebenfalls. Einen öffentlichen Update-Pfad, bekannte Probleme oder Supportfristen für EPG 9.5.1 hat Kiteworks in den geöffneten Quellen nicht veröffentlicht.

## Quellen

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Hersteller-Mitteilung vom 25. September 2026, die 9.5.1 als aktuelle Version nennt.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Offizielle Übersicht der am 30. September 2026 veröffentlichten Advisories, darunter die EPG-Einträge für 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Kritische Zugriffskontrollschwäche, CVSS 9.4, behoben in EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Nicht authentifiziertes Schreiben von Dateien, CVSS 7.5, behoben in EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Schreiben beliebiger Dateien mit Administratorrechten, CVSS 6.5, behoben in EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Nicht authentifizierter Denial of Service, CVSS 5.3, behoben in EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Herstellerangabe zur Weiterführung und Umbenennung der totemo-Technologie als Email Protection Gateway.
