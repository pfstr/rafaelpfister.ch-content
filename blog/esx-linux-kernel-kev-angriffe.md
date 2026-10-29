---
title: "VMware ESX und Linux-Kernel: CISA meldet Angriffe auf CVE-2026-41709 und Fragnesia"
navTitle: "ESX + Fragnesia (KEV)"
description: "CISA hat am 28. Oktober 2026 zwei Lücken in den KEV-Katalog aufgenommen: CVE-2026-41709 in VMware ESX (Operationen ohne Protokolleintrag, CVSS 2.7) und CVE-2026-46300 (Fragnesia) im Linux-Kernel (Rechteausweitung, CVSS 7.8). Beide wurden in realen Angriffsketten eingesetzt. Updates gibt es seit Monaten."
date: "2026-10-29"
kategorie: "Sicherheitslücken"
timeToRead: "6 Min. Lesezeit"
themen:
  - "sicherheitsluecken"
  - "linux-kernel"
produkte:
  - "sicherheitsluecken"
  - "linux-kernel"
protokolle:
  - "haertung"
hauptthema: "sicherheitsluecken"
slug: "esx-linux-kernel-kev-angriffe"
translationId: article-1a7276cb70f423c0
url: https://rafaelpfister.ch/blog/esx-linux-kernel-kev-angriffe
---

# VMware ESX und Linux-Kernel: CISA meldet Angriffe auf CVE-2026-41709 und Fragnesia

Die US-Behörde CISA hat am 28. Oktober 2026 zwei Schwachstellen in ihren Katalog der aktiv ausgenutzten Lücken (Known Exploited Vulnerabilities, KEV) aufgenommen: CVE-2026-41709 in VMware ESX und CVE-2026-46300 im Linux-Kernel, bekannt als Fragnesia. Für beide gibt es seit Monaten Updates. Angreifer haben beide Lücken mit einem klaren Zweck eingesetzt, obwohl sie nur niedrig beziehungsweise hoch und nicht kritisch bewertet sind: die eine, um Spuren zu vermeiden, die andere, um nach einem Erstzugriff Root-Rechte zu erlangen.

| CVE | Produkt | Art | CVSS | Updates seit |
|---|---|---|---|---|
| CVE-2026-41709 | VMware ESX | Operationen werden nicht protokolliert | 2.7 (niedrig) | 29.07.2026 |
| CVE-2026-46300 (Fragnesia) | Linux-Kernel, XFRM ESP-in-TCP | lokale Rechteausweitung auf root | 7.8 (hoch) | Mai 2026 |

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Unterstützung bei Prüfung und Update</p>
<p>Wenn Sie Hilfe beim Inventar betroffener ESX-Hosts oder Linux-Systeme brauchen, nutzen Sie bitte das <a href="https://adeptio.ch/">Kontaktformular auf adeptio.ch</a>. Ich melde mich auch kurzfristig.</p>
</div>

## CVE-2026-41709: ESX protokolliert nicht alle Operationen

Broadcom hat die Lücke am 29. Juli 2026 im Advisory VMSA-2026-0006 veröffentlicht, zusammen mit mehreren kritischen Lücken in ESX, vCenter, Workstation und Fusion. Laut Advisory kann ein Administrator mit bösen Absichten bestimmte Operationen auf dem ESX-Host ausführen, ohne dass diese protokolliert werden. Broadcom bewertet das mit CVSS 2.7 als niedrig, weil der Angreifer bereits Administratorrechte braucht. Einen Workaround gibt es nicht.

Die niedrige Bewertung beschreibt die Lücke isoliert. Im Angriff hat sie eine andere Funktion: Wer bereits Kontrolle über die Virtualisierungsumgebung hat, kann damit Befehle ausführen, die in der späteren Forensik fehlen. CrowdStrike beschreibt im Legal Threat Landscape Report 2026, dass die China zugeordnete Gruppe Warp Panda bei Einbrüchen in US-Organisationen, darunter Anwaltskanzleien, Schadsoftware in vCenter-Umgebungen platziert und CVE-2026-41709 in mehreren Fällen als Zero-Day ausgenutzt hat, zuletzt Anfang 2026. Die Lücke wurde also mehrere Monate vor dem Patch ausgenutzt.

Fixversionen laut Broadcom:

| Produkt | ESX-Version | Fixversion |
|---|---|---|
| VMware Cloud Foundation, vSphere Foundation | 9.1.x | ESXi-9.1.0.0-25370933 |
| VMware Cloud Foundation, vSphere Foundation | 9.0.x | ESXi-9.0.2.0100-25595025 |
| VMware ESX | 8.0 | ESXi80U3j-25429389 |
| VMware Cloud Foundation | 5.x | 5.2.4 (Async-Patching nach KB88287) |
| VMware Telco Cloud Platform | 5.0.x, 5.1.x | siehe KB449886 |

Die installierte Version und den Build eines Hosts zeigt die ESX-Shell:

```shell
esxcli system version get
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `system version get` | gibt Produkt, Version, Update-Stand und Build-Nummer des Hosts aus |

</details>

Für die Bewertung eines möglichen Vorfalls bedeutet das: Auf ESX-Hosts, die vor dem Update kompromittiert waren, sind die lokalen Protokolle nicht vollständig. Ein Abgleich mit externen Quellen (vCenter-Ereignisse, Netzwerk- und Firewall-Logs, Logs der Backup-Software) ist dann aussagekräftiger als das Host-Log allein.

## CVE-2026-46300 (Fragnesia): Root über den Page Cache

Fragnesia ist eine Variante der Lücke Dirty Frag und liegt im XFRM-ESP-Subsystem des Kernels, genauer in ESP-in-TCP (IPsec über TCP). Ein lokaler Benutzer ohne Rechte erhält damit Schreibzugriff auf den Page Cache und kann den zwischengespeicherten Inhalt von Dateien ändern, die er nur lesen darf. Der öffentliche Exploit verändert so `/usr/bin/su` und startet eine Root-Shell. William Bowling von Zellic hat die Lücke am 13. Mai 2026 mit Exploit und Patch veröffentlicht; Microsoft warnte kurz darauf. Red Hat bewertet sie mit CVSS 7.8. Die Patches für Dirty Frag beheben Fragnesia nicht, es braucht einen eigenen Kernel-Fix.

Die grossen Distributionen liefern korrigierte Kernel über die normalen Updates aus, zum Beispiel AlmaLinux 8 ab `kernel-4.18.0-553.124.2.el8_10`, AlmaLinux 9 ab `kernel-5.14.0-611.54.4.el9_7` und Ubuntu 20.04 ab `5.15.0-181.191~20.04.1`. Für andere Distributionen gilt das jeweilige Advisory; Ubuntu weist darauf hin, dass bei Backports die Versionsnummer allein wenig aussagt.

Ob die betroffenen Module geladen sind, zeigt:

```shell
lsmod | grep -E '^(esp4|esp6|rxrpc) '
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `lsmod` | listet die aktuell geladenen Kernel-Module |
| `grep -E` | erweiterte reguläre Ausdrücke, hier für die Alternative `esp4\|esp6\|rxrpc` |
| `'^(…) '` | trifft nur Zeilen, die mit genau diesem Modulnamen beginnen |

</details>

Bis ein korrigierter Kernel installiert ist, lassen sich die Module entladen und am erneuten Laden hindern. Das ist nur möglich, wenn das System kein IPsec braucht:

```shell
rmmod esp4 esp6 rxrpc
printf 'install esp4 /bin/false\ninstall esp6 /bin/false\ninstall rxrpc /bin/false\n' > /etc/modprobe.d/dirtyfrag.conf
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `rmmod esp4 esp6 rxrpc` | entlädt die drei Module aus dem laufenden Kernel |
| `install <modul> /bin/false` | Eintrag in modprobe.d: beim Laden des Moduls wird `/bin/false` ausgeführt statt das Modul geladen |
| `> /etc/modprobe.d/dirtyfrag.conf` | schreibt die Regeln in eine eigene Konfigurationsdatei; bestehende Dateien bleiben unverändert |

</details>

## Die Angriffskette über php2shell

Wie Fragnesia in der Praxis eingesetzt wird, beschreibt Dream Research in einer Analyse vom 26. Oktober 2026. Ein Angreifer hat zwischen dem 8. und 24. Oktober 25 Firewall-Server in den USA, Japan, Taiwan und Südkorea übernommen. Den Hersteller nennt Dream nicht. Die Kette bestand aus zwei Schritten in einer einzigen HTTP-Anfrage:

1. **Erstzugriff über php2shell:** Die beiden PHP-Lücken CVE-2026-109618 und CVE-2026-109619 erlauben Codeausführung auf dem Management-Webserver der Firewall, im beschriebenen Fall als Benutzer `nobody`.
2. **Rechteausweitung über Fragnesia:** SELinux verhinderte, dass der Webserver-Prozess Systemdateien ändert. Der Angreifer überschrieb deshalb über den Page Cache `/etc/pam.d/su` mit `pam_permit.so` und erhielt mit `su` ohne Passwort Root-Rechte.

Auf einer Firewall der Zielorganisation richtete der Angreifer danach einen dauerhaften SSH-Reverse-Tunnel ein, übernahm den internen Management-Server, entschlüsselte das Passwort des LDAP-Dienstkontos und griff das Active Directory an. Die übrigen 22 Firewalls blieben mit einer Webshell versehen. Dream ordnet die Kampagne einem Akteur mit China-Bezug zu und sieht Hinweise auf KI-gestützte Entwicklung der rund 40 Skripte, ohne direkten Beleg.

Eine lokale Rechteausweitung ist damit auch auf Appliances relevant: Eine Webanwendung mit eingeschränkten Rechten reicht als Ausgangspunkt, wenn der Kernel ungepatcht ist.

## php2shell und PAN-OS

PHP hat die php2shell-Lücken am 21. Oktober 2026 geschlossen (PHP 8.5.12, 8.4.27, 8.3.36 und 8.2.35); CISA führt beide seit dem 26. Oktober im KEV-Katalog. Palo Alto Networks meldet aktive Angriffe auf das Management-Webinterface von PAN-OS über diese Lücken (Advisory CVE-2026-109618, CVSS 4.0 9.2). Betroffen sind PA-Series- und VM-Series-Firewalls sowie Panorama; GlobalProtect und Captive Portal sind laut Hersteller nicht betroffen. Für die unterstützten Versionen 10.2 bis 12.2 sind Hotfixes erschienen; für 11.1.7-h12 nennt das Advisory den 30. Oktober 2026. Das Risiko ist am höchsten, wenn das Management-Interface aus dem Internet erreichbar ist; Palo Alto empfiehlt, den Zugriff auf vertrauenswürdige interne Adressen zu beschränken.

## Was jetzt zu tun ist

1. ESX-Hosts auf die Fixversionen aus VMSA-2026-0006 bringen. Das Advisory schliesst zugleich die kritischen Lücken CVE-2026-59309, CVE-2026-59310 und CVE-2026-47876.
2. Linux-Systeme auf einen Kernel mit Fragnesia-Fix aktualisieren und neu starten. Das betrifft auch Appliances auf Linux-Basis, für die der Hersteller ein eigenes Update liefern muss.
3. PHP auf die Versionen vom 21. Oktober 2026 oder einen Distributions-Backport aktualisieren; bei PAN-OS die Hotfixes aus dem Advisory einspielen.
4. Management-Schnittstellen von Firewalls, Hypervisoren und vCenter nicht aus dem Internet erreichbar machen.
5. Bei Verdacht auf Kompromittierung von ESX-Hosts die Protokolle mit externen Quellen abgleichen, weil das Host-Log vor dem Update unvollständig sein kann.

## Quellen

1.  [CISA: CISA Adds Two Known Exploited Vulnerabilities to Catalog (28.10.2026)](https://www.cisa.gov/news-events/alerts/2026/10/28/cisa-adds-two-known-exploited-vulnerabilities-catalog): Aufnahme von CVE-2026-41709 und CVE-2026-46300 in den KEV-Katalog.

2.  [Broadcom: VMSA-2026-0006](https://support.broadcom.com/web/ecx/support-content-notification/-/external/content/SecurityAdvisories/0/38017): Advisory mit Beschreibung von CVE-2026-41709, Response-Matrix und Fixversionen.

3.  [CrowdStrike: 2026 Legal Threat Landscape Report](https://www.crowdstrike.com/en-us/blog/crowdstrike-2026-legal-threat-landscape-report/): Warp Panda in vCenter-Umgebungen und Ausnutzung von CVE-2026-41709 als Zero-Day.

4.  [Dream Research: Chained php2shell exploit roots 25 servers](https://dreamgroup.com/blog/chained-php2shell-exploit-roots-25-servers): Analyse der Angriffskette aus php2shell und Fragnesia mit Artefakten und Zuordnung.

5.  [Palo Alto Networks: CVE-2026-109618](https://security.paloaltonetworks.com/CVE-2026-109618): Advisory zu den Auswirkungen von php2shell auf PAN-OS mit betroffenen Versionen, Hotfixes und Mitigationen.

6.  [CISA: CISA Adds Two Known Exploited Vulnerabilities to Catalog (26.10.2026)](https://www.cisa.gov/news-events/alerts/2026/10/26/cisa-adds-two-known-exploited-vulnerabilities-catalog): Aufnahme der php2shell-Lücken CVE-2026-109618 und CVE-2026-109619.

7.  [Ubuntu: Fragnesia Linux kernel local privilege escalation vulnerability mitigations](https://ubuntu.com/blog/fragnesia-linux-vulnerability-fixes-available): betroffene Ubuntu-Releases, Fixversionen und Modul-Sperre als Übergangslösung.

8.  [Tenable: Fragnesia (CVE-2026-46300) FAQ](https://tenable.com/blog/fragnesia-cve-2026-46300-faq-about-new-linux-kernel-xfrm-esp-in-tcp-priv-esc): Hintergrund zu ESP-in-TCP, Abgrenzung zu Dirty Frag und Fixstand der Distributionen.

9.  [heise online: Angriffe auf Linux- und VMware-ESX-Schwachstellen](https://www.heise.de/news/Angriffe-auf-Linux-und-VMware-ESX-Schwachstellen-11503947.html): deutschsprachige Meldung mit Einordnung der niedrigen Broadcom-Bewertung.

10.  [heise online: Fragnesia: Microsoft warnt vor weiterer Rechteausweitungslücke in Linux](https://www.heise.de/news/Fragnesia-Microsoft-warnt-vor-weiterer-Rechteausweitungsluecke-in-Linux-11294817.html): Details zu Entdeckung, Exploit und Workaround.
