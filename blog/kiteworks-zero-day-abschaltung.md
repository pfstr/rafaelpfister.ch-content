---
title: "Kiteworks: Hersteller empfiehlt Abschaltung am 26. September, was das für den Mailflow bedeutet"
navTitle: "Kiteworks-Abschaltung"
description: "Kiteworks fordert seine Kunden per E-Mail auf, alle Systeme am Samstag, 26.09.2026, von 04:00 bis 10:00 Uhr herunterzufahren. Grund ist eine Warnung von Strafverfolgungsbehörden vor einem Zero-Day-Angriff. Hängt Kiteworks als Verschlüsselungsgateway im Mailflow, stoppt die Abschaltung den Mailverkehr, wenn er nicht vorher umgeleitet wird."
date: "2026-09-25"
kategorie: "Totemomail"
timeToRead: "2 Min. Lesezeit"
themen:
  - "totemomail"
  - "smtp-mailflow"
  - "e-mail-verschluesselung"
produkte:
  - "totemomail"
  - "exchange-online"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
slug: "kiteworks-zero-day-abschaltung"
featured: "2026-09-27"
warnung: true
translationId: "article-38fbaa0e9095957a"
url: "https://rafaelpfister.ch/blog/kiteworks-zero-day-abschaltung"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks (Email Protection Gateway, ehemals totemomail) empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
---
# Kiteworks: Hersteller empfiehlt Abschaltung am 26. September, was das für den Mailflow bedeutet

Kiteworks hat seine Kunden am 25. September 2026 per E-Mail aufgefordert, alle Kiteworks-Systeme am Samstag, 26. September, von 04:00 bis 10:00 Uhr (mitteleuropäische Zeit) herunterzufahren. Laut dem Schreiben des CISO Frank Balonis liegen dem Hersteller Hinweise von Strafverfolgungsbehörden vor, dass an diesem Wochenende ein Angriff auf Kiteworks-Systeme bevorstehen könnte. Der Kundensupport begründet die Abschaltung mit dem Schutz vor möglichen Zero-Day-Angriffen. heise online hat die Echtheit der Nachricht telefonisch beim Support bestätigt.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Notfall-Unterstützung beim Umschalten des Mailflows</p>
<p>Wenn Sie Hilfe brauchen, um den Mailflow vor der Abschaltung umzuleiten und danach wieder zurückzustellen, erreichen Sie mich auch kurzfristig unter <a href="tel:+41585102208">+41 58 510 22 08</a>.</p>
</div>

## Was bekannt ist

Die Empfehlung gilt weltweit; die E-Mail nennt das Zeitfenster für alle Zeitzonen von AEST bis PDT. Kiteworks rät, die Systeme schon vor Beginn des Fensters herunterzufahren, und zwar auch dann, wenn sie nicht aus dem Internet erreichbar sind.

Offen ist bisher fast alles andere: Es gibt kein öffentliches Security Advisory, keine CVE-Nummer, keinen Patch und keine Angabe dazu, welche Produkte oder Versionen betroffen sind. Welche Behörde die Warnung ausgesprochen hat, ist ebenfalls nicht bekannt. Auf den offiziellen Kanälen von Kiteworks (Security Updates, Newsroom, GitHub-Advisories) finde ich Stand 25. September keinen Eintrag dazu. Die einzige Quelle ist die Kunden-E-Mail, die nicht öffentlich einsehbar ist.

## Quellen

1.  [heise online: Bevorstehender Zero-Day-Angriff: KiteWorks drängt Kunden zur Serverabschaltung](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Erstmeldung mit Auszügen aus der Kunden-E-Mail und dem Zeitfenster.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): englische Fassung mit dem Originalwortlaut des CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): offizieller Kanal des Herstellers, Stand 25.09.2026 ohne Eintrag zur Warnung.

4.  [Kiteworks: Security Advisories auf GitHub](https://github.com/kiteworks/security-advisories/security): Advisory-Liste des Herstellers, letzter Eintrag vom 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offizielle Mitteilungen, Stand 25.09.2026 ohne Eintrag zur Warnung.
