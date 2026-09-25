---
title: "Kiteworks: Hersteller empfiehlt Abschaltung am 26. September - Was bislang bekannt ist"
navTitle: "Kiteworks-Abschaltung"
description: "Kiteworks fordert seine Kunden per E-Mail auf, alle Systeme am Samstag, 26.09.2026, von 04:00 bis 10:00 Uhr herunterzufahren. Grund ist eine Warnung von Strafverfolgungsbehörden vor einem möglichen Angriff. Totemomail ist nicht betroffen."
date: "2026-09-25"
kategorie: "Totemomail"
timeToRead: "6 Min. Lesezeit"
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
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
---
# Kiteworks: Hersteller empfiehlt Abschaltung am 26. September - Was bislang bekannt ist

Kiteworks hat seine Kunden am 25. September 2026 per E-Mail aufgefordert, alle Kiteworks-Systeme am Samstag, 26. September, von 04:00 bis 10:00 Uhr (mitteleuropäische Zeit) herunterzufahren. Laut dem Schreiben des CISO Frank Balonis liegen dem Hersteller Hinweise von Strafverfolgungsbehörden vor, dass an diesem Wochenende ein Angriff auf Kiteworks-Systeme bevorstehen könnte. Der Kundensupport begründet die Abschaltung mit dem Schutz vor möglichen Zero-Day-Angriffen. heise online hat die Echtheit der Nachricht telefonisch beim Support bestätigt.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Notfall-Unterstützung beim Umschalten des Mailflows</p>
<p>Wenn Sie Hilfe brauchen, um den Mailflow vor der Abschaltung umzuleiten und danach wieder zurückzustellen, erreichen Sie mich auch kurzfristig unter <a href="tel:+41585102208">+41 58 510 22 08</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update vom 25. September 2026: Stellungnahme von Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail ist nicht betroffen.</strong> Offen ist, ob Kiteworks EPG (Email Protection Gateway) betroffen ist.</p>
</div>

## Was bekannt ist

Die Empfehlung gilt weltweit; die E-Mail nennt das Zeitfenster für alle Zeitzonen von AEST bis PDT. Kiteworks rät, die Systeme schon vor Beginn des Fensters herunterzufahren, und zwar auch dann, wenn sie nicht aus dem Internet erreichbar sind.

Offen ist bisher fast alles andere: Es gibt kein öffentliches Security Advisory, keine CVE-Nummer, keinen Patch und keine Angabe dazu, welche Produkte oder Versionen betroffen sind. Welche Behörde die Warnung ausgesprochen hat, ist ebenfalls nicht bekannt. Auf den offiziellen Kanälen von Kiteworks (Security Updates, Newsroom, GitHub-Advisories) finde ich Stand 25. September keinen Eintrag dazu. Die einzige Quelle ist die Kunden-E-Mail, die nicht öffentlich einsehbar ist.

## Mögliche Ursachen: Theorien

Solange Kiteworks keine Details veröffentlicht, bleibt die Ursache offen. Die folgenden Erklärungen sind Hypothesen, die sich aus den bekannten Eckdaten ableiten lassen; einige davon werden auch in den Kommentaren zur heise-Meldung diskutiert. Keine davon ist bestätigt.

Zwei Eckdaten schränken den Raum ein. Erstens nennt die Warnung ein festes Zeitfenster von sechs Stunden statt einer unbefristeten Abschaltung bis zum Patch. Zweitens sollen auch Systeme vom Netz, die nicht aus dem Internet erreichbar sind. Eine klassische, über das Internet ausnutzbare Lücke würde weder das eine noch das andere erklären: Dagegen hilft es, das System vom Internet zu trennen, und zwar so lange, bis der Patch da ist.

### 1. Den Behörden ist ein geplanter Zeitpunkt bekannt

Strafverfolgungsbehörden erfahren den Zeitpunkt einer geplanten Kampagne gelegentlich vorab, etwa aus überwachter Kommunikation einer Tätergruppe oder aus beschlagnahmter Infrastruktur. Massenhafte Ausnutzungen von Dateiaustausch-Produkten laufen typischerweise in einem kurzen, koordinierten Fenster ab, oft an Wochenenden oder Feiertagen, wenn weniger Personal im Einsatz ist. Kiteworks ist der Nachfolger von Accellion, dessen File Transfer Appliance 2020 und 2021 genau so angegriffen wurde: Über mehrere Lücken wurden Daten abgezogen, anschliessend wurden die betroffenen Organisationen erpresst.

Dafür spricht das eng umrissene Fenster am Wochenende. Dagegen spricht der Einwand, den mehrere Kommentatoren bei heise erheben: Die Warnung ging an alle Kunden, die Angreifer dürften also davon wissen und können den Angriff einfach verschieben. Eine Verschiebung würde dem Hersteller aber Zeit für einen Patch verschaffen.

### 2. Der Hersteller kennt die Lücke selbst noch nicht

Möglich ist auch, dass Kiteworks ausser dem Hinweis der Behörden keine technischen Details hat, also weder die betroffene Komponente kennt noch einen Patch oder eine Konfigurationsänderung empfehlen kann. Dann ist die Abschaltung die einzige Massnahme, die ohne Kenntnis der Lücke wirkt, und das feste Ende ein Kompromiss, den die Kunden eher mittragen. In den heise-Kommentaren wird dazu die Vermutung geäussert, der Hersteller könnte während des Fensters einzelne Systeme als Köder online lassen, um den Angriff zu beobachten. Belege dafür gibt es keine.

Dafür spricht, dass weder ein Advisory noch eine Mitigation genannt wird. Dagegen spricht, dass Kiteworks nach eigenen Angaben mit Mandiant zusammenarbeitet und bei einer Warnung von Behörden in der Regel zumindest Indikatoren zur Verfügung stehen.

### 3. Eine bereits platzierte Hintertür mit Zeitauslöser

Die Empfehlung, auch interne Systeme herunterzufahren, passt zu einem Szenario, in dem der Angriff nicht von aussen kommt, sondern auf den Appliances bereits vorbereitet ist: etwa eine Hintertür aus einer früheren Kompromittierung, die zu einem festen Zeitpunkt aktiv wird oder Kontakt zu einem Kontrollserver aufnimmt. Ein ausgeschaltetes System kann zu diesem Zeitpunkt nichts ausführen.

Dafür spricht, dass die Erreichbarkeit aus dem Internet bei diesem Szenario keine Rolle spielt. Dagegen spricht, dass ein Hersteller in diesem Fall eher eine Prüfung auf Kompromittierung und eine Neuinstallation empfehlen würde als einen Neustart nach sechs Stunden.

### 4. Kompromittierung auf Herstellerseite

Ein weiterer Weg, der interne Systeme erreicht, sind Verbindungen, die von der Appliance zum Hersteller aufgebaut werden, etwa für Updates, Lizenzprüfung oder Fernwartung. Ist ein solcher Kanal kompromittiert, schützt eine Firewall vor eingehendem Verkehr nicht. Die Abschaltung würde dem Hersteller in diesem Szenario ein Fenster verschaffen, um die eigene Infrastruktur zu bereinigen, Schlüssel oder Zertifikate zu tauschen und erst danach wieder Verbindungen zuzulassen.

Dafür spricht der weltweit einheitliche Zeitpunkt, der zu einer koordinierten Aktion auf Herstellerseite passt. Dagegen spricht, dass der Hersteller dann eher empfehlen würde, die ausgehenden Verbindungen zu sperren, statt die Systeme ganz abzuschalten.

### 5. Begleitmassnahme zu einer Behördenaktion

Denkbar ist schliesslich, dass die Behörden im selben Zeitraum gegen die Infrastruktur der Angreifer vorgehen und verhindern wollen, dass diese als Reaktion noch schnell zuschlagen. Das würde das kurze Fenster und die Rolle der Strafverfolgung erklären. Für diese Variante gibt es bisher keinerlei öffentliche Hinweise.

### Kritik an der Kommunikation

In den heise-Kommentaren überwiegt Skepsis, und die Einwände sind sachlich nachvollziehbar: Ohne Angaben zur Lücke lässt sich nicht beurteilen, ob eine Trennung vom Internet per Firewall genügt hätte. Ein Zeitfenster ohne angekündigten Patch lässt offen, was nach 10:00 Uhr gilt. Und eine Warnung, die nur per E-Mail an Kunden geht, erreicht nicht alle Betreiber, etwa bei Partnern, Dienstleistern oder nach Personalwechseln. Unabhängig davon, welche Theorie zutrifft: Wer Kiteworks betreibt, sollte nach dem Wiederhochfahren die Protokolle prüfen und die Kanäle des Herstellers beobachten, bis ein Advisory vorliegt.

## Quellen

1.  [heise online: Bevorstehender Zero-Day-Angriff: KiteWorks drängt Kunden zur Serverabschaltung](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Erstmeldung mit Auszügen aus der Kunden-E-Mail und dem Zeitfenster.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): englische Fassung mit dem Originalwortlaut des CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): offizieller Kanal des Herstellers, Stand 25.09.2026 ohne Eintrag zur Warnung.

4.  [Kiteworks: Security Advisories auf GitHub](https://github.com/kiteworks/security-advisories/security): Advisory-Liste des Herstellers, letzter Eintrag vom 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offizielle Mitteilungen, Stand 25.09.2026 ohne Eintrag zur Warnung.

6.  [heise-Forum: Kommentare zur Meldung](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Leserdiskussion mit den Einwänden zum festen Zeitfenster und zur Abschaltung interner Systeme sowie der Köder-These.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory zur Ausnutzung der Accellion FTA 2020/2021 mit anschliessender Erpressung; Accellion ist der frühere Name von Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Herstellerangabe zur Zusammenarbeit mit Mandiant.
