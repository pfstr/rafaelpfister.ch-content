---
title: "Kiteworks: Hersteller empfiehlt Abschaltung am 26. September, was das für den Mailflow bedeutet"
navTitle: "Kiteworks-Abschaltung"
description: "Kiteworks fordert seine Kunden per E-Mail auf, alle Systeme am Samstag, 26.09.2026, von 04:00 bis 10:00 Uhr herunterzufahren. Grund ist eine Warnung von Strafverfolgungsbehörden vor einem Zero-Day-Angriff. Hängt Kiteworks als Verschlüsselungsgateway im Mailflow, stoppt die Abschaltung den Mailverkehr, wenn er nicht vorher umgeleitet wird."
date: "2026-09-25"
kategorie: "Totemomail"
timeToRead: "4 Min. Lesezeit"
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

## Folgen für den Mailflow

Das Kiteworks Email Protection Gateway (ehemals totemomail) steht in vielen Umgebungen direkt im Mailflow, häufig in einer Mail-Schlaufe mit Exchange Online (Aufbau siehe [Totemomail mit Microsoft 365](/blog/totemomail-m365)). In diesem Fall lässt sich das Gateway nicht abschalten, ohne den Mailverkehr zu unterbrechen:

- **Ausgehend:** Leiten ein Outbound-Connector oder Transportregeln Mails über das Gateway, stellt Exchange sie in die Warteschlange. Wie lange Exchange die Zustellung wiederholt und ab wann Verzögerungs- oder Unzustellbarkeitsmeldungen an die Absender gehen, hängt von der Umgebung ab.
- **Eingehend:** Zeigt der MX-Eintrag auf das Gateway, wiederholen die sendenden Server die Zustellung nach ihren eigenen Regeln. Nach RFC 5321 sollten sie das über mehrere Tage tun; verlassen können Sie sich bei sechs Stunden Ausfall darauf aber nicht bei jedem Absender.
- **Schlaufe:** Kommen eingehende Mails über Exchange Online zum Gateway und wieder zurück, betrifft die Abschaltung beide Richtungen, auch Mails, die gar nicht verschlüsselt werden müssen.

## Vor der Abschaltung klären

1.  Welche Connectoren, Transportregeln und MX-Einträge leiten Mails über Kiteworks, ein- und ausgehend?
2.  Soll der Mailflow für die Dauer der Abschaltung am Gateway vorbeilaufen, oder reicht es, ihn in der Warteschlange zu halten?
3.  Welche Mails dürfen auf keinen Fall unverschlüsselt hinausgehen? Für diese braucht es bei einer Umleitung eine Regel, die sie zurückhält, statt sie ohne Verschlüsselung zuzustellen.
4.  Wer informiert Benutzer und Partner, die über Kiteworks sichere Mails, Dateien oder Formulare austauschen?
5.  Wie wird der ursprüngliche Zustand nach 10:00 Uhr wiederhergestellt? Die geänderten Einstellungen vorher dokumentieren, damit sich jeder Schritt gezielt zurücknehmen lässt.

Nach dem Neustart lohnt sich ein Blick auf die Protokolle der letzten Tage (Anmeldungen, Admin-Zugriffe, ungewöhnliche Downloads) und auf die Kanäle des Herstellers, bevor die Systeme wieder produktiv gehen. Sobald Kiteworks ein Advisory oder einen Patch veröffentlicht, ergänze ich diesen Artikel.

## Quellen

1.  [heise online: Bevorstehender Zero-Day-Angriff: KiteWorks drängt Kunden zur Serverabschaltung](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Erstmeldung mit Auszügen aus der Kunden-E-Mail und dem Zeitfenster.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): englische Fassung mit dem Originalwortlaut des CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): offizieller Kanal des Herstellers, Stand 25.09.2026 ohne Eintrag zur Warnung.

4.  [Kiteworks: Security Advisories auf GitHub](https://github.com/kiteworks/security-advisories/security): Advisory-Liste des Herstellers, letzter Eintrag vom 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offizielle Mitteilungen, Stand 25.09.2026 ohne Eintrag zur Warnung.

6.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321#section-4.5.4.1): Abschnitt 4.5.4.1 zu Wiederholungsintervallen und Aufgabezeit beim Zustellversuch.
