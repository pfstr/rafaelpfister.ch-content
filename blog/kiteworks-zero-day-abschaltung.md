---
title: "Kiteworks: Hersteller empfiehlt Abschaltung am 26. September - Was bislang bekannt ist"
navTitle: "Kiteworks-Abschaltung"
description: "Kiteworks forderte seine Kunden auf, alle Systeme am Samstag, 26.09.2026, von 04:00 bis 10:00 Uhr herunterzufahren. Abschlussbericht: Während der Abschaltung fand und schloss der Hersteller eine kritische Lücke ohne CVE; am 30.09. folgten 125 Advisories, darunter CVE-2026-54154 (CVSS 10.0) im Email Protection Gateway. Totemomail ist nicht betroffen."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 Min. Lesezeit"
themen:
  - "totemomail"
  - "sicherheitsluecken"
produkte:
  - "totemomail"
  - "sicherheitsluecken"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-zero-day-abschaltung"
featured: "2026-09-28"
translationId: "article-38fbaa0e9095957a"
url: "https://rafaelpfister.ch/blog/kiteworks-zero-day-abschaltung"
aiPrompt: |
  Du bist mein Assistent für Kiteworks-Betrieb und Mail-Sicherheit. Kiteworks hat am 30.09.2026 125 Security Advisories für Core, Email Protection Gateway, Secure Data Forms und MFT veröffentlicht, alle behoben bis Version 9.5.1, darunter CVE-2026-54154 (CVSS 10.0, Email Protection Gateway vor 9.4.1). Hilf mir festzustellen, welche Kiteworks-Komponenten ich betreibe und in welcher Version, welche Advisories mich betreffen, wie ich das Update auf 9.5.1 plane (Cluster-Reihenfolge, Wartungsmodus, Mailflow-Umleitung während des Updates) und welche Protokolle ich auf Hinweise einer Kompromittierung prüfe. Frage zuerst nach meinem Aufbau (selbst betrieben oder gehostet, Komponenten, Version, Position des Gateways im Mailflow).
---
# Kiteworks: Hersteller empfiehlt Abschaltung am 26. September - Was bislang bekannt ist

Kiteworks hat seine Kunden am 25. September 2026 per E-Mail aufgefordert, alle Kiteworks-Systeme am Samstag, 26. September, von 04:00 bis 10:00 Uhr (mitteleuropäische Zeit) herunterzufahren. Laut dem Schreiben des CISO Frank Balonis lagen dem Hersteller Hinweise von Strafverfolgungsbehörden vor, dass an diesem Wochenende ein Angriff auf Kiteworks-Systeme bevorstehen könnte. Der Kundensupport begründete die Abschaltung mit dem Schutz vor möglichen Zero-Day-Angriffen. heise online hat die Echtheit der Nachricht telefonisch beim Support bestätigt.

<div class="update-hinweis">
<p class="update-hinweis__titel">Abschlussbericht vom 7. Oktober 2026</p>
<p>Der Vorfall ist aus Sicht des Herstellers abgeschlossen. Die Abschaltempfehlung gilt seit dem 27. September nicht mehr, ein Angriff auf Kiteworks- oder Kundensysteme ist bis heute nicht bekannt geworden. Die wichtigsten Ergebnisse:</p>
<ul>
<li><strong>Kritische Lücke während der Abschaltung gefunden:</strong> Laut Pressemitteilung vom 28. September stiess Kiteworks bei der Analyse mit den Bundesbehörden auf eine bisher unbekannte kritische Schwachstelle in einer Funktion, die bei weniger als 1 % der Kunden aktiviert ist. Der Hersteller hat im Abschaltfenster einen Fix entwickelt und ausgerollt und zusätzlich eine Schutzschicht in allen Umgebungen aktiviert. Welche Funktion betroffen war, ist nicht veröffentlicht; eine CVE-Nummer gibt es dafür bis heute nicht.</li>
<li><strong>125 Advisories am 30. September:</strong> Zwei Tage später veröffentlichte Kiteworks auf GitHub 125 Security Advisories für Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) und MFT Server (3); 12 davon kritisch, 49 hoch. Alle sind in Versionen bis einschliesslich 9.5.1 behoben, die meisten wurden über das Bug-Bounty-Programm auf YesWeHack gemeldet. Mit der Lücke aus dem Abschaltfenster haben sie nach heutigem Stand nichts zu tun.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> Die schwerste Lücke betrifft das Email Protection Gateway vor Version 9.4.1. Ein nicht authentifizierter Angreifer kann über öffentlich erreichbare Endpunkte Code mit Root-Rechten ausführen. 10 der 12 kritischen Advisories betreffen das Email Protection Gateway.</li>
<li><strong>Keine bekannte Ausnutzung:</strong> Für keine der Lücken gibt es Berichte über Angriffe; im CISA-KEV-Katalog steht Stand 7. Oktober kein Kiteworks-Eintrag aus 2026. Shadowserver zählt laut BleepingComputer knapp 400 aus dem Internet erreichbare Kiteworks-Instanzen.</li>
<li><strong>Totemomail:</strong> Taucht in keinem der Advisories auf und war laut Hersteller von der Abschaltung nicht betroffen.</li>
</ul>
<p><strong>Handlungsbedarf:</strong> Wer Kiteworks selbst betreibt, sollte alle Komponenten auf Version 9.5.1 bringen; das Email Protection Gateway hat dabei Vorrang. Die Details stehen im Abschnitt <a href="#abschlussbericht">Abschlussbericht</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update vom 28. September 2026: Kiteworks hebt die Abschaltempfehlung auf</p>
<p>Kiteworks hat die Pressemitteilung um einen Hinweis ergänzt: Seit dem 27. September gilt die Empfehlung zur Abschaltung für alle Kunden nicht mehr.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Wer seine Systeme noch nicht wieder hochgefahren hat, kann dies jetzt tun. Wer Advanced Forms selbst betreibt, soll sich vor dem Neustart an den Kiteworks-Support wenden. Die von Kiteworks gehosteten Instanzen laufen wieder. Weiterhin gibt es keine CVE-Nummer, keine neue Version über 9.5.1 hinaus, keine Indikatoren für eine Kompromittierung und keine Angabe, ob ein Angriff versucht wurde oder was hinter der Warnung stand. Security-Updates-Seite und GitHub-Advisories sind unverändert.</p>
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
<li class="timeline__item">
<p class="timeline__zeit">Sa, 26. September, 10:00</p>
<p class="timeline__titel">Ende des Fensters</p>
<p>Das in der Kunden-E-Mail genannte Fenster endet. Die formelle Aufhebung der Empfehlung folgt am 27. September.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">So, 27. September</p>
<p class="timeline__titel">Empfehlung aufgehoben</p>
<p>Kiteworks ergänzt die Pressemitteilung: Die Abschaltempfehlung ist für alle Kunden aufgehoben, die Systeme dürfen wieder laufen. Die gehosteten Instanzen sind wieder in Betrieb. Kunden mit selbst betriebenen Advanced Forms sollen sich an den Support wenden.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mo, 28. September</p>
<p class="timeline__titel">Kritische Lücke gefunden und geschlossen</p>
<p>Kiteworks meldet in einer weiteren Pressemitteilung, dass bei der Arbeit mit den Bundesbehörden während der Abschaltung eine bisher unbekannte kritische Schwachstelle gefunden wurde. Sie betrifft eine Funktion, die bei weniger als 1 % der Kunden aktiviert ist. Fix und zusätzliche Schutzschicht sind ausgerollt, Hinweise auf eine Kompromittierung gibt es keine. Funktion und CVE-Nummer nennt der Hersteller nicht.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mi, 30. September, ab 18:38</p>
<p class="timeline__titel">125 Security Advisories auf GitHub</p>
<p>Kiteworks veröffentlicht 125 Advisories für Core, Email Protection Gateway, Secure Data Forms und MFT Server, alle behoben bis Version 9.5.1. Die schwerste Lücke ist CVE-2026-54154 im Email Protection Gateway vor 9.4.1 (CVSS 10.0, Codeausführung mit Root-Rechten ohne Anmeldung).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Do, 1. Oktober</p>
<p class="timeline__titel">Medienberichte und MS-ISAC-Advisory</p>
<p>BleepingComputer, SecurityOnline und weitere berichten über die Advisories; das MS-ISAC (Center for Internet Security) gibt ein eigenes Advisory zu CVE-2026-54154 heraus. Eine Ausnutzung ist nicht bekannt.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Stand Mi, 7. Oktober</p>
<p class="timeline__titel">Abschluss</p>
<p>Keine Berichte über einen erfolgten oder versuchten Angriff, kein Kiteworks-Eintrag im CISA-KEV-Katalog. Offen bleiben die betroffene Funktion, eine CVE-Nummer für die Lücke aus dem Abschaltfenster und der Hintergrund der Behördenwarnung.</p>
</li>
</ol>

## Was bekannt ist

Die Empfehlung gilt weltweit; die E-Mail nennt das Zeitfenster für alle Zeitzonen von AEST bis PDT. Kiteworks rät, die Systeme schon vor Beginn des Fensters herunterzufahren, und zwar auch dann, wenn sie nicht aus dem Internet erreichbar sind.

Bis zum 28. September war fast alles andere offen: Es gab kein öffentliches Security Advisory, keine CVE-Nummer, keinen Patch und keine Angabe dazu, welche Produkte oder Versionen betroffen sind. Die Pressemitteilung nennt als Quelle „federal intelligence authorities“, vermutlich also US-Bundesbehörden; welche, ist bis heute nicht bekannt. In den GitHub-Advisories von Kiteworks stammte der letzte Eintrag bis dahin vom 27. Mai 2026; die Advisories vom 30. September sind im Abschnitt [Abschlussbericht](#abschlussbericht) zusammengefasst.

Gegenüber TechCrunch hat Kiteworks-CISO Frank Balonis die Stellungnahme im selben Wortlaut abgegeben. Das BKA hat gegenüber heise eine Stellungnahme aus ermittlungstaktischen Gründen abgelehnt, das BSI hat nicht geantwortet. Das FBI wollte sich gegenüber TechCrunch nicht äussern, ein Sprecher der CISA wollte sich nicht öffentlich äussern. Ein Kunde aus dem Gesundheitswesen hat laut TechCrunch seinen Server sofort vom Netz genommen, mit spürbaren Einschränkungen im Betrieb: Ärzte konnten ihre Patienten zeitweise nur verzögert erreichen. Laut einem von TechCrunch zitierten Sicherheitsforscher sind mindestens 1000 Kiteworks-Systeme aus dem Internet erreichbar; BornCity spricht von mehr als 1000 Organisationen, die die Warnung erhalten haben.

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

Dieser Abschnitt entstand vor dem 28. September; die Einordnung nach dem heutigen Stand steht im [Abschlussbericht](#abschlussbericht). Die folgenden Erklärungen sind Hypothesen, die sich aus den bekannten Eckdaten ableiten lassen; einige davon werden auch in den Kommentaren zur heise-Meldung diskutiert. Keine davon ist bestätigt.

Drei Eckdaten schränken den Raum ein. Erstens nennt die Warnung ein festes Zeitfenster statt einer unbefristeten Abschaltung bis zum Patch. Zweitens sollen auch Systeme vom Netz, die nicht aus dem Internet erreichbar sind. Drittens liegt das Fenster weltweit auf derselben Uhrzeit (02:00 bis 08:00 UTC) statt jeweils in der lokalen Nacht. Eine klassische, über das Internet ausnutzbare Lücke würde die ersten beiden Punkte nicht erklären: Dagegen hilft es, das System vom Internet zu trennen, und zwar so lange, bis der Patch da ist.

### 1. Den Behörden ist ein geplanter Zeitpunkt bekannt

Strafverfolgungsbehörden erfahren den Zeitpunkt einer geplanten Kampagne gelegentlich vorab, etwa aus überwachter Kommunikation einer Tätergruppe oder aus beschlagnahmter Infrastruktur. Massenhafte Ausnutzungen von Dateiaustausch-Produkten laufen typischerweise in einem kurzen, koordinierten Fenster ab, oft an Wochenenden oder Feiertagen, wenn weniger Personal im Einsatz ist. Kiteworks ist der Nachfolger von Accellion, dessen File Transfer Appliance 2020 und 2021 genau so angegriffen wurde, damals der Gruppe Clop zugeordnet: Über mehrere Lücken wurden Daten abgezogen, anschliessend wurden die betroffenen Organisationen erpresst.

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

## Abschlussbericht

Stand 7. Oktober 2026 ist der Vorfall aus Sicht des Herstellers abgeschlossen. Die Ereignisse nach dem Abschaltfenster lassen sich in zwei Stränge trennen: die Lücke, die während der Abschaltung gefunden wurde, und die Sammelveröffentlichung von Advisories zwei Tage später.

### Die Lücke aus dem Abschaltfenster

Am 28. September veröffentlichte Kiteworks eine zweite Pressemitteilung. Danach hat der Hersteller das Wochenende über mit den Bundesbehörden zusammengearbeitet; dabei wurde eine bisher unbekannte kritische Schwachstelle entdeckt, die auf eine Funktion beschränkt ist, die bei weniger als 1 % der Kunden aktiviert ist. Kiteworks hat im Abschaltfenster einen Fix entwickelt und ausgerollt und zusätzlich eine Schutzschicht in allen Umgebungen aktiviert. Die durchgehende Überwachung habe keine auffälligen Aktivitäten gezeigt, es gebe keinen Hinweis auf eine Kompromittierung von Kiteworks- oder Kundensystemen. Alle übrigen Kiteworks-Produkte seien nicht betroffen.

Nicht veröffentlicht sind die betroffene Funktion, eine CVE-Nummer, die Versionen mit dem Fix und die Frage, ob selbst betriebene Installationen den Fix automatisch erhalten haben. Die Aufhebung vom 27. September enthielt nur eine Ausnahme: Kunden mit selbst betriebenen Advanced Forms sollten vor dem Neustart den Support kontaktieren. Ob diese Funktion die betroffene war, hat Kiteworks nicht bestätigt.

Zu den Theorien weiter oben: Die Pressemitteilung beschreibt eine Lücke, die erst während des Fensters gefunden wurde. Das passt zu Theorie 2 (der Hersteller kannte die Lücke vorher nicht) in Verbindung mit Theorie 1 (die Behörden kannten einen geplanten Zeitpunkt). Für die Theorien 3 bis 5 gibt es keine Bestätigung. Was die Behörden konkret wussten und ob ein Angriff versucht wurde, ist weiterhin nicht bekannt.

### 125 Security Advisories vom 30. September

Am 30. September ab 18:38 Uhr veröffentlichte Kiteworks auf GitHub 125 Security Advisories auf einen Schlag. Sie verteilen sich wie folgt:

| Produkt | Advisories | davon kritisch |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Gesamt** | **125** | **12** |

Nach Schweregrad sind es 12 kritische, 49 hohe, 52 mittlere und 12 niedrige Einstufungen. Alle Lücken sind in Versionen bis einschliesslich 9.5.1 behoben; die ältesten Einträge betreffen Version 9.2.1. Es handelt sich also um eine nachträgliche Offenlegung bereits ausgelieferter Fixes, nicht um eine neue Version. Als Melder nennen die Advisories überwiegend Teilnehmer des Bug-Bounty-Programms auf YesWeHack. Eine Verbindung zur Lücke aus dem Abschaltfenster stellt Kiteworks nicht her; eine Woche nach dem Fenster entspricht der Stand der Advisories weiterhin der Aussage vom 25. September, dass alle bekannten Lücken in 9.5.1 behoben sind.

Die kritischen Advisories:

| CVE | Produkt | CVSS 3.1 | behoben ab | Auswirkung |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Codeausführung mit Root-Rechten ohne Anmeldung |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Kontoübernahme |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Kontoübernahme |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Kontoübernahme über die Passwort-Zurücksetzung |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Kontoübernahme |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Kontoübernahme |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Umgehung von Sicherheitsfunktionen |
| CVE-2026-102095, CVE-2026-102102 bis 102105 | EPG | 9.1 | 9.5.0 | Zugriff auf interne Netzwerkressourcen (SSRF) |

CVE-2026-54154 ist die schwerste Lücke: Laut Advisory ermöglicht eine Kombination von Fehlern bei der Eingabeprüfung in öffentlich erreichbaren Endpunkten des Email Protection Gateway einem nicht angemeldeten Angreifer, Code auszuführen und über weitere lokale Schwächen Root-Rechte auf der Appliance zu erlangen. Das MS-ISAC hat dazu am 1. Oktober ein eigenes Advisory herausgegeben. Für Mail-Administratoren ist das Gateway der relevante Teil der Veröffentlichung: Es steht typischerweise direkt im Mailfluss und ist aus dem Internet erreichbar.

### Ausnutzung und Verbreitung

Für keine der Lücken liegen Berichte über eine Ausnutzung oder öffentliche Exploits vor. Der CISA-KEV-Katalog enthält Stand 7. Oktober nur die vier Accellion-FTA-Einträge aus 2021. Shadowserver zählt laut BleepingComputer knapp 400 aus dem Internet erreichbare Kiteworks-Instanzen; wie viele davon bereits auf 9.5.1 laufen, ist nicht bekannt. Totemomail kommt in keinem der Advisories vor.

## Nach dem Fenster: Was Betreiber jetzt tun können

Die Abschaltempfehlung ist aufgehoben, die bekannten Lücken sind in Version 9.5.1 behoben. Für selbst betriebene Installationen sind folgende Schritte sinnvoll:

1.  **Version prüfen:** Läuft auf allen Knoten und allen Komponenten (Core, Email Protection Gateway, Secure Data Forms, MFT Server) Version 9.5.1? Ein Email Protection Gateway vor 9.4.1 ist von CVE-2026-54154 betroffen und sollte zuerst aktualisiert werden.

2.  **Cluster-Zustand prüfen:** Im Cluster Health Dashboard sollten alle Knoten grün sein und der Wartungsmodus ausgeschaltet.

3.  **Protokolle auswerten:** Anmeldungen, Administrator-Aktionen und ungewöhnliche Datei-Downloads um das Abschaltfenster herum sichten, insbesondere bei Systemen, die nicht oder erst spät heruntergefahren wurden.

4.  **Erreichbarkeit einschränken:** Wo möglich, den Zugriff aus dem Internet auf die Administrationsoberfläche sperren und nur benötigte Dienste freigeben.

5.  **Advanced Forms:** Wer das Modul selbst betreibt und noch keinen Kontakt mit dem Support hatte, klärt mit Kiteworks, ob der Fix aus dem Abschaltfenster auf der eigenen Installation angekommen ist.

6.  **Advisories abgleichen:** Die GitHub-Advisories lassen sich nach Produkt filtern (Präfix `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Für jede eingesetzte Komponente prüfen, ob die installierte Version unter der jeweils genannten Fix-Version liegt.

7.  **Kanäle beobachten:** GitHub-Advisories, Newsroom und Kunden-E-Mails von Kiteworks, falls der Hersteller zur Lücke aus dem Abschaltfenster doch noch ein Advisory mit CVE-Nummer veröffentlicht. Neue CVEs zum Kiteworks Email Protection Gateway und zu Totemomail führt auch der [CVE-Tracker](/cve) dieser Seite; dort lässt sich eine Warn-Mail abonnieren.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Unterstützung beim Update</p>
<p>Wenn Sie Hilfe beim Update eines Kiteworks- oder Totemomail-Gateways brauchen, etwa bei der Umleitung des Mailflows während des Wartungsfensters oder bei der Auswertung der Protokolle, nutzen Sie bitte das <a href="https://adeptio.ch/">Kontaktformular auf adeptio.ch</a>.</p>
</div>

## Quellen

1.  [heise online: Bevorstehender Zero-Day-Angriff: KiteWorks drängt Kunden zur Serverabschaltung](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Erstmeldung mit Auszügen aus der Kunden-E-Mail und dem Zeitfenster; Update vom 25.09., 17:41 Uhr, mit der Antwort des BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): englische Fassung mit dem Originalwortlaut des CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): ältere Update-Seite des Herstellers, Stand 07.10.2026 ohne Eintrag zur Warnung oder zu den Advisories vom 30.09.2026.

4.  [Kiteworks: Security Advisories auf GitHub](https://github.com/kiteworks/security-advisories/security): Advisory-Liste des Herstellers, bis 28.09.2026 letzter Eintrag vom 27.05.2026; am 30.09.2026 125 neue Advisories für Core, EPG, SDF und MFT. Zahlen in diesem Artikel per GitHub-API ausgezählt.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offizielle Mitteilungen, seit 25.09.2026 mit der Pressemitteilung zur Abschaltung.

6.  [heise-Forum: Kommentare zur Meldung](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Leserdiskussion mit den Einwänden zum festen Zeitfenster und zur Abschaltung interner Systeme sowie der Köder-These.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory zur Ausnutzung der Accellion FTA 2020/2021 mit anschliessender Erpressung; Accellion ist der frühere Name von Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Herstellerangabe zur Zusammenarbeit mit Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): Stellungnahme des CISO, Versandzeit der Warnung, Reaktionen von FBI und CISA (Nachtrag), Auswirkungen bei einem Kunden, Zahl der aus dem Internet erreichbaren Systeme.

10.  [Kiteworks: Precautionary Shutdown Advisory (Pressemitteilung)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): offizielle Mitteilung vom 25.09.2026 mit Angaben zu gehosteten Instanzen, Version 9.5.1 und den nicht betroffenen Tochterfirmen; ergänzt um den Hinweis vom 27.09.2026, dass die Abschaltempfehlung aufgehoben ist.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Zeitfenster nach Regionen und Einordnung früherer Angriffe auf Dateiaustausch-Produkte.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): Einschätzung von watchTowr zur ungewöhnlichen Abschaltempfehlung.

13.  [BornCity: Kiteworks: Mehr als 1.000 Organisationen sollen Server abschalten](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): Zahl der benachrichtigten Organisationen und Branchen im deutschsprachigen Raum.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): Einordnung der Accellion-Angriffe durch Clop 2020/2021 und Zitat von watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): Bericht vom 28.09.2026 über die Aufhebung der Empfehlung und den Betrieb der gehosteten Instanzen.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (Pressemitteilung)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): Mitteilung vom 28.09.2026 zur während der Abschaltung gefundenen kritischen Lücke, zum Fix und zur zusätzlichen Schutzschicht.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): Zusammenfassung der zweiten Pressemitteilung mit Zitaten des CISO.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Herstellerangaben zur Codeausführung im Email Protection Gateway vor 9.4.1, CVSS 10.0, Meldung über YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): Bericht vom 01.10.2026 zu CVE-2026-54154 und Zahl der von Shadowserver gezählten Instanzen.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): Advisory des Center for Internet Security vom 01.10.2026 mit Empfehlungen.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): Einordnung der Kontoübernahme-Lücken in Core, darunter CVE-2026-102115; die Zählung weicht von der Advisory-Liste ab.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): Stand 07.10.2026 nur die vier Accellion-FTA-Einträge aus 2021, kein Kiteworks-Eintrag aus 2026.
