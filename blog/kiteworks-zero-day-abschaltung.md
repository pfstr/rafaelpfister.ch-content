---
title: "Kiteworks: Hersteller empfiehlt Abschaltung am 26. September - Was bislang bekannt ist"
navTitle: "Kiteworks-Abschaltung"
description: "Kiteworks fordert seine Kunden per E-Mail auf, alle Systeme am Samstag, 26.09.2026, von 04:00 bis 10:00 Uhr herunterzufahren. Grund ist eine Warnung von Strafverfolgungsbehörden vor einem möglichen Angriff. Totemomail ist nicht betroffen."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 Min. Lesezeit"
themen:
  - "totemomail"
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
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
<p>Wenn Sie Hilfe brauchen, um den Mailflow vor der Abschaltung umzuleiten und danach wieder zurückzustellen, nutzen Sie bitte das <a href="https://adeptio.ch/">Kontaktformular auf adeptio.ch</a>. Ich melde mich auch kurzfristig.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update vom 25. September 2026: Stellungnahme von Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail ist nicht betroffen.</strong> Offen ist, ob Kiteworks EPG (Email Protection Gateway) betroffen ist.</p>
</div>

## Chronologie

Alle Zeiten in mitteleuropäischer Sommerzeit (MESZ). Wo keine Uhrzeit angegeben ist, liegt keine belastbare Zeitangabe vor.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fr, 25. September</p>
<p class="timeline__titel">Advisory an die Kunden</p>
<p>CISO Frank Balonis informiert die Kunden per E-Mail über Hinweise von Strafverfolgungsbehörden auf einen möglichen Angriff an diesem Wochenende und empfiehlt eine Abschaltung für sechs Stunden. Laut Advisory sind alle bekannten Schwachstellen in Version 9.5.1 behoben.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fr, 25. September</p>
<p class="timeline__titel">Erste Medienberichte</p>
<p>heise online berichtet, der Kiteworks-Support bestätigt die Echtheit der Nachricht und begründet die Abschaltung mit dem Schutz vor möglichen Zero-Day-Angriffen. Kurz darauf folgen TechCrunch, BleepingComputer, Computer Weekly und weitere.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fr, 25. September, 17:41</p>
<p class="timeline__titel">BKA äussert sich nicht</p>
<p>heise ergänzt: Das BKA lehnt eine Stellungnahme aus ermittlungstaktischen Gründen ab. Das BSI antwortet nicht, das FBI verzichtet gegenüber TechCrunch auf einen Kommentar.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fr, 25. September</p>
<p class="timeline__titel">Stellungnahme und Pressemitteilung</p>
<p>Kiteworks bezeichnet die Abschaltung als Vorsichtsmassnahme ohne bekannte Kompromittierung. Die Pressemitteilung nennt „federal intelligence authorities“ als Quelle und listet die nicht betroffenen Tochterfirmen auf, darunter totemo. Von Kiteworks gehostete Instanzen fährt der Hersteller selbst herunter.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sa, 26. September, 04:00 bis 10:00</p>
<p class="timeline__titel">Abschaltfenster</p>
<p>Das Fenster liegt weltweit gleichzeitig: 02:00 bis 08:00 UTC, in Sydney 12:00 bis 18:00, in New York Freitag 22:00 bis Samstag 04:00.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Stand Sa, 26. September</p>
<p class="timeline__titel">Weiterhin offen</p>
<p>Kein öffentliches Advisory, keine CVE-Nummer, keine Angaben zur Lücke und keine Berichte über einen erfolgten Angriff.</p>
</li>
</ol>

## Was bekannt ist

Die Empfehlung gilt weltweit; die E-Mail nennt das Zeitfenster für alle Zeitzonen von AEST bis PDT. Kiteworks rät, die Systeme schon vor Beginn des Fensters herunterzufahren, und zwar auch dann, wenn sie nicht aus dem Internet erreichbar sind.

Offen ist bisher fast alles andere: Es gibt kein öffentliches Security Advisory, keine CVE-Nummer, keinen Patch und keine Angabe dazu, welche Produkte oder Versionen betroffen sind. Die Pressemitteilung nennt als Quelle „federal intelligence authorities“, vermutlich also US-Bundesbehörden; welche, ist nicht bekannt. Unter Security Updates und in den GitHub-Advisories von Kiteworks gibt es Stand 26. September keinen Eintrag. Öffentlich sind die oben zitierte Stellungnahme und die Pressemitteilung vom 25. September.

Gegenüber TechCrunch hat Kiteworks-CISO Frank Balonis die Stellungnahme im selben Wortlaut abgegeben. Das BKA hat gegenüber heise eine Stellungnahme aus ermittlungstaktischen Gründen abgelehnt, das BSI hat nicht geantwortet. Das FBI wollte sich gegenüber TechCrunch nicht äussern, von der CISA lag keine Antwort vor. Ein Kunde aus dem Gesundheitswesen hat laut TechCrunch seinen Server sofort vom Netz genommen, mit spürbaren Einschränkungen im Betrieb.

Die Pressemitteilung weicht in einem Punkt von der Kunden-E-Mail ab: Sie spricht von einem Abschaltfenster von neun Stunden, die Advisory an die Kunden von sechs Stunden. Laut Pressemitteilung betrifft die Empfehlung nur selbst betriebene Installationen (On-Premises, AWS, Azure). Nicht betroffen sind nach Angaben des Herstellers die Tochterfirmen Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai und 123FormBuilder.

## Die Advisory an die Kunden

Die Kunden-E-Mail vom 25. September enthält neben der Warnung einen Zeitplan pro Zeitzone und eine Anleitung für Cluster. Rechnet man die Zeiten auf UTC um, ergibt sich für alle Regionen dasselbe Fenster von 02:00 bis 08:00 UTC.

| Zeitzone | Stadt | Beginn | Ende |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Sa, 12:00 | Sa, 18:00 |
| SGT (UTC+8) | Singapur | Sa, 10:00 | Sa, 16:00 |
| IDT (UTC+3) | Tel Aviv | Sa, 05:00 | Sa, 11:00 |
| CEST (UTC+2) | Amsterdam, Zürich | Sa, 04:00 | Sa, 10:00 |
| BST (UTC+1) | London | Sa, 03:00 | Sa, 09:00 |
| EDT (UTC−4) | New York | Fr, 22:00 | Sa, 04:00 |
| CDT (UTC−5) | Chicago | Fr, 21:00 | Sa, 03:00 |
| MDT (UTC−6) | Denver | Fr, 20:00 | Sa, 02:00 |
| PDT (UTC−7) | San Francisco | Fr, 19:00 | Sa, 01:00 |

Für Cluster mit mehreren Servern gibt Kiteworks eine feste Reihenfolge vor:

1.  **Wartungsmodus einschalten** unter System Setup > Maintenance Mode, damit keine Benutzer mehr zugreifen.

2.  **Sicherung erstellen:** einen Snapshot jedes Knotens oder ein Backup der Kiteworks-Datenbank (System Setup > Cluster Configuration > System Configuration). Es wird nur ein Datenbank-Backup vorgehalten; jedes neue ersetzt das vorherige.

3.  **Rollen erfassen:** Unter System Setup > Locations zeigt die Spalte Assigned Roles, welche Knoten die Application-Rolle haben; der primäre Application-Knoten ist mit einem Stern markiert. Die Knoten und ihre IP-Adressen notieren, sie werden für den Neustart gebraucht.

4.  **Herunterfahren in dieser Reihenfolge:** zuerst alle Knoten ohne Application-Rolle, dann die übrigen Application-Knoten, zuletzt den primären Application-Knoten. Das geht über den Reiter Shut Down des jeweiligen Knotens oder über die Konsole des Hypervisors (etwa VMware oder AWS), wenn die Kiteworks-Oberfläche nicht mehr erreichbar ist.

5.  **Neustart in umgekehrter Reihenfolge** über den Hypervisor, da die Admin-Konsole erst erreichbar ist, wenn genügend Knoten laufen (Anhang E des Administrator Guide): zuerst den primären Application-Knoten, dann die übrigen Application-Knoten einzeln und jeweils erst, wenn der vorherige vollständig läuft, damit die Datenbank-Server ein Quorum bilden können. Danach die Storage-Server, dann die übrigen Rollen (Repositories Gateway, Search, SFTP, Antivirus), zuletzt die Web-Server.

6.  **Wartungsmodus ausschalten**, sobald alle Knoten im Cluster Health Dashboard auf der Statusseite der Admin-Konsole grün sind.

Auf Nachfrage hat der Kiteworks-Support zudem bestätigt, dass keine der Tochterfirmen von Kiteworks betroffen ist.

## Mögliche Ursachen: Theorien

Solange Kiteworks keine Details veröffentlicht, bleibt die Ursache offen. Die folgenden Erklärungen sind Hypothesen, die sich aus den bekannten Eckdaten ableiten lassen; einige davon werden auch in den Kommentaren zur heise-Meldung diskutiert. Keine davon ist bestätigt.

Drei Eckdaten schränken den Raum ein. Erstens nennt die Warnung ein festes Zeitfenster statt einer unbefristeten Abschaltung bis zum Patch. Zweitens sollen auch Systeme vom Netz, die nicht aus dem Internet erreichbar sind. Drittens liegt das Fenster weltweit auf derselben Uhrzeit (02:00 bis 08:00 UTC) statt jeweils in der lokalen Nacht. Eine klassische, über das Internet ausnutzbare Lücke würde die ersten beiden Punkte nicht erklären: Dagegen hilft es, das System vom Internet zu trennen, und zwar so lange, bis der Patch da ist.

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

Denkbar ist schliesslich, dass die Behörden im selben Zeitraum gegen die Infrastruktur der Angreifer vorgehen und verhindern wollen, dass diese als Reaktion noch schnell zuschlagen. Das würde das kurze Fenster und die Rolle der Strafverfolgung erklären. Dass das BKA eine Stellungnahme aus ermittlungstaktischen Gründen ablehnt, deutet auf laufende Ermittlungen hin, belegt diese Variante aber nicht.

### Kritik an der Kommunikation

In den heise-Kommentaren überwiegt Skepsis, und die Einwände sind sachlich nachvollziehbar: Ohne Angaben zur Lücke lässt sich nicht beurteilen, ob eine Trennung vom Internet per Firewall genügt hätte. Ein Zeitfenster ohne angekündigten Patch lässt offen, was nach 10:00 Uhr gilt. Und eine Warnung, die nur per E-Mail an Kunden geht, erreicht nicht alle Betreiber, etwa bei Partnern, Dienstleistern oder nach Personalwechseln. Unabhängig davon, welche Theorie zutrifft: Wer Kiteworks betreibt, sollte nach dem Wiederhochfahren die Protokolle prüfen und die Kanäle des Herstellers beobachten, bis ein Advisory vorliegt.

## Quellen

1.  [heise online: Bevorstehender Zero-Day-Angriff: KiteWorks drängt Kunden zur Serverabschaltung](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Erstmeldung mit Auszügen aus der Kunden-E-Mail und dem Zeitfenster; Update vom 25.09., 17:41 Uhr, mit der Antwort des BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): englische Fassung mit dem Originalwortlaut des CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): offizieller Kanal des Herstellers, Stand 26.09.2026 ohne Eintrag zur Warnung.

4.  [Kiteworks: Security Advisories auf GitHub](https://github.com/kiteworks/security-advisories/security): Advisory-Liste des Herstellers, letzter Eintrag vom 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offizielle Mitteilungen, seit 25.09.2026 mit der Pressemitteilung zur Abschaltung.

6.  [heise-Forum: Kommentare zur Meldung](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Leserdiskussion mit den Einwänden zum festen Zeitfenster und zur Abschaltung interner Systeme sowie der Köder-These.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory zur Ausnutzung der Accellion FTA 2020/2021 mit anschliessender Erpressung; Accellion ist der frühere Name von Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Herstellerangabe zur Zusammenarbeit mit Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): Stellungnahme des CISO, Versandzeit der Warnung, Reaktionen von FBI und CISA, Auswirkungen bei einem Kunden.

10.  [Kiteworks: Precautionary Shutdown Advisory (Pressemitteilung)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): offizielle Mitteilung vom 25.09.2026 mit Angaben zu gehosteten Instanzen, Version 9.5.1 und den nicht betroffenen Tochterfirmen.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Zeitfenster nach Regionen und Einordnung früherer Angriffe auf Dateiaustausch-Produkte.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): Einschätzung von watchTowr zur ungewöhnlichen Abschaltempfehlung.
