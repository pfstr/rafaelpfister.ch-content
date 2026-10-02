---
title: "FortiMail: Zero-Day-Lücke CVE-2026-104286 wird ausgenutzt - Workaround und Prüfung auf Kompromittierung"
navTitle: "FortiMail Zero-Day"
description: "Fortinet meldet eine kritische, bereits ausgenutzte Path-Traversal-Lücke in FortiMail 7.2 bis 8.0 (CVE-2026-104286, CVSS 9.8). Ein Patch fehlt bislang. Der Workaround deaktiviert IBE oder sperrt die Verwaltungsoberfläche gegen das Internet; dazu kommen Indikatoren für eine Prüfung auf Kompromittierung."
date: "2026-10-02"
kategorie: "FortiMail"
timeToRead: "8 Min. Lesezeit"
themen:
  - "fortimail"
  - "e-mail-verschluesselung"
produkte:
  - "fortimail"
protokolle:
  - "haertung"
  - "verschluesselung"
hauptthema: "fortimail"
slug: "fortimail-zero-day-workaround"
featured: "2026-10-02"
warnung: true
translationId: "article-0f2b7855c8ece9fe"
url: "https://rafaelpfister.ch/blog/fortimail-zero-day-workaround"
aiPrompt: |
  Du bist mein Assistent für Fortinet FortiMail. Für CVE-2026-104286 (FG-IR-26-175) empfiehlt Fortinet als Workaround, IBE zu deaktivieren oder die Verwaltungsoberfläche gegen das Internet zu sperren. Hilf mir Schritt für Schritt: Version und IBE-Status auf meinen FortiMail-Systemen ermitteln, abschätzen, welche Richtlinien und Empfänger von einer IBE-Abschaltung betroffen sind, den Workaround umsetzen und die von Fortinet genannten Indikatoren (Dateien, IP-Adressen, Logeinträge) prüfen. Frage zuerst nach Version, Betriebsmodus (Gateway, Server, Transparent), Erreichbarkeit der Weboberfläche aus dem Internet und danach, ob IBE produktiv genutzt wird.
---
# FortiMail: Zero-Day-Lücke CVE-2026-104286 wird ausgenutzt - Workaround und Prüfung auf Kompromittierung

Fortinet hat am 1. Oktober 2026 das Advisory FG-IR-26-175 zu einer kritischen Lücke in FortiMail veröffentlicht. Über präparierte HTTP- oder HTTPS-Anfragen kann ein Angreifer ohne Anmeldung beliebige Dateien auf das darunterliegende System schreiben und damit Code ausführen. Die Lücke trägt die Kennung CVE-2026-104286 und eine CVSS-Bewertung von 9.8. Fortinet bestätigt, dass sie bereits ausgenutzt wird; die US-Behörde CISA hat sie am selben Tag in ihren Katalog ausgenutzter Schwachstellen (KEV) aufgenommen. Ein Update gibt es Stand 2. Oktober noch nicht, nur einen Workaround.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Unterstützung bei Workaround und Prüfung</p>
<p>Wenn Sie Hilfe beim Umsetzen des Workarounds, bei der Prüfung auf Kompromittierung oder beim späteren Update brauchen, nutzen Sie bitte das <a href="https://adeptio.ch/">Kontaktformular auf adeptio.ch</a>. Ich melde mich auch kurzfristig.</p>
</div>

<div class="zertifikat-hinweis">
<img src="/images/fortimail-7-4-administrator-badge.webp" alt="Badge Fortinet FortiMail 7.4 Administrator" width="88" height="88" loading="lazy">
<div>
<p class="zertifikat-hinweis__titel">Zertifiziert: Fortinet FortiMail 7.4 Administrator</p>
<p>Ich bin von Fortinet als FortiMail 7.4 Administrator zertifiziert. Die Prüfung umfasst Bereitstellung, Verwaltung, laufenden Betrieb und Fehlersuche von FortiMail. <a href="https://www.credly.com/badges/90a9e1d5-1f57-4762-99cc-4b35a9079ca5">Badge auf Credly prüfen</a></p>
</div>
</div>

## Das Wichtigste in Kürze

| Merkmal | Angabe |
|---|---|
| Advisory | FG-IR-26-175, veröffentlicht am 1. Oktober 2026 |
| CVE | CVE-2026-104286 |
| Bewertung | CVSS 3.1: 9.8 (kritisch), `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| Art der Lücke | Path Traversal (CWE-22) und unzureichend gefilterte NULL-Zeichen (CWE-158) |
| Voraussetzung | keine Anmeldung, Zugriff per HTTP oder HTTPS auf die Weboberfläche |
| Komponente laut Advisory | GUI |
| Ausnutzung | bestätigt; CISA KEV seit 1. Oktober 2026 |
| Update | angekündigt, Stand 2. Oktober nicht verfügbar |
| Workaround | IBE deaktivieren oder Verwaltungsoberfläche gegen das Internet sperren |
| Entdeckt durch | Fortinet intern (Gwendal Guégniaud) |

## Chronologie

Alle Zeiten in mitteleuropäischer Sommerzeit (MESZ). Wo keine Uhrzeit angegeben ist, liegt keine belastbare Zeitangabe vor.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Mi, 1. Oktober</p>
<p class="timeline__titel">Advisory FG-IR-26-175</p>
<p>Fortinet veröffentlicht das Advisory mit betroffenen Versionen, Workaround und Indikatoren. Die Ausnutzung ist zu diesem Zeitpunkt bereits bekannt; einen virtuellen Patch für FortiGate-IPS gibt es laut Advisory nicht.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mi, 1. Oktober, 08:46</p>
<p class="timeline__titel">Bericht bei heise online</p>
<p>heise berichtet über die laufenden Angriffe und den Workaround; Updates zum Schliessen der Lücke stehen laut Bericht noch aus.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mi, 1. Oktober</p>
<p class="timeline__titel">Aufnahme in den CISA-Katalog</p>
<p>CISA nimmt CVE-2026-104286 in den Katalog Known Exploited Vulnerabilities auf. US-Bundesbehörden sollen laut BleepingComputer und The Hacker News bis zum 4. Oktober Updates oder Workarounds umsetzen.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Stand Do, 2. Oktober</p>
<p class="timeline__titel">Weiterhin offen</p>
<p>Die Fixversionen 8.0.2, 7.6.7 und 7.4.9 sind angekündigt, aber nicht veröffentlicht. Zum Umfang der Angriffe, zu betroffenen Organisationen und zur Herkunft der Angreifer gibt es keine Angaben.</p>
</li>
</ol>

## Betroffene Versionen

| Zweig | Betroffen | Lösung laut Advisory |
|---|---|---|
| FortiMail 8.0 | 8.0.0 bis 8.0.1 | Update auf 8.0.2 oder neuer (angekündigt) |
| FortiMail 7.6 | 7.6.0 bis 7.6.6 | Update auf 7.6.7 oder neuer (angekündigt) |
| FortiMail 7.4 | 7.4.0 bis 7.4.8 | Update auf 7.4.9 oder neuer (angekündigt) |
| FortiMail 7.2 | 7.2.0 bis 7.2.9 | kein Fix im Zweig 7.2; Wechsel auf 7.4 oder neuer |

Für den Zweig 7.2 erscheint kein Update; diese Systeme müssen auf 7.4 oder neuer wechseln. Den Upgrade-Pfad aus den Release Notes jetzt zu planen verkürzt die Zeit bis zum Update, sobald die Fixversion erscheint. Bis dahin bleibt nur der Workaround.

Die aktuelle Version zeigt die CLI mit:

```
get system status
```

## Sofortmassnahme: Workaround umsetzen

Fortinet nennt zwei Varianten. Eine davon genügt laut Advisory; wo es der Betrieb zulässt, sind beide sinnvoll.

### Variante 1: IBE deaktivieren

IBE (Identity-Based Encryption) ist die Funktion, mit der FortiMail Nachrichten an externe Empfänger verschlüsselt zustellt: Der Empfänger erhält die Nachricht als verschlüsselten Anhang (Push) oder ruft sie über das Webportal von FortiMail ab (Pull). Fortinet nennt die IBE-Funktion als Workaround, die Angriffe laufen also über einen Codepfad dieser Funktion. Die Abschaltung erfolgt in der CLI:

```
config system encryption ibe
    set status disable
end
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `config system encryption ibe` | öffnet die globalen IBE-Einstellungen |
| `set status disable` | schaltet IBE systemweit aus |
| `end` | speichert die Änderung und verlässt den Konfigurationsabschnitt |

</details>

Den Zustand vorher und nachher prüfen Sie mit:

```
show system encryption ibe
```

Die Abschaltung hat Folgen für den Mailverkehr. Fortinet beschreibt sie im Advisory nicht. Vor der Änderung sollten Sie deshalb klären:

1.  **Welche Richtlinien IBE nutzen:** In Content- und Policy-Profilen ist IBE als Verschlüsselungsaktion hinterlegt, etwa bei Betreff-Schlüsselwörtern wie `[secure]` oder bei Datenschutzregeln. Diese Nachrichten können während der Abschaltung nicht mehr per IBE verschlüsselt zugestellt werden.

2.  **Wie diese Nachrichten behandelt werden sollen:** zurückhalten, mit einem anderen Verfahren (S/MIME, TLS-Pflicht) zustellen oder den Absendern eine Rückmeldung geben. Eine unverschlüsselte Zustellung von Nachrichten, die eine Richtlinie verschlüsseln soll, ist in der Regel nicht zulässig.

3.  **Wer betroffen ist:** Externe Empfänger, die Nachrichten im IBE-Portal abrufen, und Benutzer, die verschlüsselt versenden, sollten informiert werden.

### Variante 2: Verwaltungsoberfläche gegen das Internet sperren

Die zweite Variante sperrt den Zugriff aus dem Internet auf die Verwaltungsoberfläche oder beschränkt ihn auf vertrauenswürdige private Netze, etwa ein Managementnetz oder ein VPN. Umsetzen lässt sich das auf einer vorgelagerten Firewall oder über die Zugriffseinstellungen der Schnittstellen in FortiMail (Administrative Access).

Webmail, Quarantäne-Zugriff der Benutzer und das IBE-Portal laufen ebenfalls über HTTPS. Liegen sie auf derselben Schnittstelle wie die Verwaltung, sperrt eine pauschale HTTPS-Sperre auch diese Dienste. In diesem Fall ist Variante 1 der direktere Weg. SMTP auf Port 25 ist von beiden Varianten nicht betroffen; der Mailfluss läuft weiter.

## Auf Kompromittierung prüfen

Da die Lücke vor dem Advisory ausgenutzt wurde, schliesst der Workaround einen früheren Angriff nicht aus. Fortinet veröffentlicht im Advisory Indikatoren, mit denen sich ein System prüfen lässt.

### Dateien

Folgende Dateien hat Fortinet auf kompromittierten Systemen gefunden. Das Advisory nennt dazu Hashwerte; massgebend für den Abgleich ist die Liste im Advisory.

| Pfad |
|---|
| `/data/lib/liblog.so` |
| `/data/etc/ld.so.preload` |
| `/bin/smit` |
| `/data/bin/webconsole` |
| `/data/bin/mailservice` |
| `/data/etc/httpd.conf` |
| `/data/migadmin.tar.gz` |

Ein Eintrag in `/data/etc/ld.so.preload` lädt die angegebene Bibliothek in jeden Prozess vor. Zusammen mit `liblog.so` deutet das auf eine dauerhafte Hintertür hin, die einen Neustart übersteht.

### IP-Adressen

| Adresse |
|---|
| `79.141.169.187` |
| `45.129.0.192` |

Diese Adressen sollten in den Logs von FortiMail, vorgelagerter Firewall und Reverse Proxy gesucht und bis auf Weiteres gesperrt werden.

### Logeinträge

Das Advisory nennt drei Muster in den Systemlogs von FortiMail:

1.  **Cron-Einträge** des Benutzers `root`, die eine Shell mit Bezug auf `/migadmin` starten, etwa `type=event subtype=system pri=debug user=system ui=cron msg="(root) CMD (/bin/sh -c 'O=/migadmin...`.

2.  **Konfiguration von Archivkonten** mit der Remote-IP `79.141.169.187`. Ein Archivkonto kann Kopien von Nachrichten an ein externes Ziel ablegen.

3.  **Base64-Dekodierfehler im IBE-Decrypter** und auffällige fehlgeschlagene Anmeldeversuche.

Treffer bei einem dieser Indikatoren bedeuten, dass das System als kompromittiert gilt. Vor jeder Änderung sollten Logs und Konfiguration gesichert werden, damit eine forensische Auswertung möglich bleibt. Ein Update allein entfernt eine eingerichtete Hintertür nicht. Zur Bereinigung gehören eine Neuinstallation aus vertrauenswürdiger Quelle, das Einspielen einer geprüften Konfiguration und der Wechsel aller Zugangsdaten, die auf dem System gespeichert waren: Administratorkonten, LDAP-Bind-Konten, SMTP-Authentifizierung und private Schlüssel von Zertifikaten. Archivkonten und Weiterleitungen sollten auf unbekannte Ziele geprüft werden, weil darüber Nachrichteninhalte abfliessen können.

## Einordnung

Mail-Gateways stehen am Rand des Netzes und verarbeiten den gesamten Nachrichtenverkehr einer Organisation. Eine Lücke, die ohne Anmeldung Code ausführt, gibt einem Angreifer Zugriff auf Nachrichteninhalte, gespeicherte Zugangsdaten und Schlüsselmaterial. Bei FortiMail ist es nicht der erste Fall: Im Mai 2025 betraf CVE-2025-32756 (FG-IR-25-254, CVSS 9.6) neben FortiVoice auch FortiMail; ausgenutzt wurde die Lücke damals nachweislich auf FortiVoice-Systemen.

Ende September hat zudem Kiteworks seine Kunden wegen einer Warnung von Strafverfolgungsbehörden aufgefordert, ihre Systeme vorsorglich herunterzufahren; die Hintergründe sind dort weiterhin offen ([Kiteworks: Abschaltempfehlung am 26. September](/blog/kiteworks-zero-day-abschaltung)). Ein Zusammenhang zwischen beiden Fällen ist nicht bekannt.

## Was jetzt zu tun ist

1.  **Version feststellen** mit `get system status` und mit der Tabelle der betroffenen Versionen abgleichen.

2.  **Indikatoren prüfen** und Logs sichern, bevor Änderungen am System erfolgen.

3.  **Workaround umsetzen:** IBE deaktivieren oder die Verwaltungsoberfläche gegen das Internet sperren, nach Abklärung der Folgen für verschlüsselte Zustellungen.

4.  **IP-Adressen sperren** und in den Firewall-Logs rückwirkend suchen.

5.  **Update planen:** Fixversion einspielen, sobald sie erscheint; Systeme auf 7.2 auf 7.4 oder neuer heben.

6.  **Advisory beobachten:** Fortinet ergänzt FG-IR-26-175 bei neuen Erkenntnissen; die Revisionshistorie steht am Ende des Advisorys.

## Quellen

1.  [Fortinet PSIRT: FG-IR-26-175](https://fortiguard.fortinet.com/psirt/FG-IR-26-175): Advisory mit Beschreibung, CVSS-Vektor, betroffenen Versionen, Workaround und Indikatoren (Dateien, Hashes, IP-Adressen, Logeinträge).

2.  [heise online: FortiMail: Angriffe auf Zero-Day-Lücke laufen, Workaround verfügbar](https://www.heise.de/news/FortiMail-Angriffe-auf-Zero-Day-Luecke-laufen-Workaround-verfuegbar-11473599.html): Meldung vom 1. Oktober 2026 mit Workaround und Hinweis auf fehlende Updates.

3.  [CISA: CISA Adds One Known Exploited Vulnerability to Catalog](https://www.cisa.gov/news-events/alerts/2026/10/01/cisa-adds-one-known-exploited-vulnerability-catalog): Aufnahme von CVE-2026-104286 in den KEV-Katalog am 1. Oktober 2026.

4.  [BleepingComputer: Fortinet warns of critical FortiMail flaw exploited in zero-day attacks](https://www.bleepingcomputer.com/news/security/fortinet-warns-of-critical-fortimail-flaw-exploited-in-zero-day-attacks/): Zusammenfassung mit angekündigten Fixversionen und Frist für US-Bundesbehörden.

5.  [The Hacker News: Critical FortiMail Zero-Day Flaw Exploited in Attacks](https://thehackernews.com/2026/10/critical-fortimail-zero-day-flaw.html): Zusammenfassung des Advisorys mit Indikatoren.

6.  [watchTowr: Fortinet FortiMail FAQ: CVE-2026-104286](https://watchtowr.com/intelligence/fortinet-fortimail-cve-2026-104286-faq/): Einordnung des IBE-Codepfads und Empfehlung, Beweise vor der Bereinigung zu sichern.

7.  [Fortinet PSIRT: FG-IR-25-254](https://fortiguard.fortinet.com/psirt/FG-IR-25-254): Advisory zu CVE-2025-32756 (Mai 2025), Stack-Overflow in FortiVoice, FortiMail und weiteren Produkten.

8.  [Fortinet Document Library: FortiMail](https://docs.fortinet.com/product/fortimail): Release Notes und Administration Guide, darunter die CLI-Referenz zu `system encryption ibe`.

9.  [Credly: Fortinet FortiMail 7.4 Administrator](https://www.credly.com/badges/90a9e1d5-1f57-4762-99cc-4b35a9079ca5): Verifikation der Zertifizierung des Autors.
