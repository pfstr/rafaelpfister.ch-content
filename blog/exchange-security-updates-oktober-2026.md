---
title: "Exchange-Sicherheitsupdates vom Oktober 2026: sechs Schwachstellen, letztes Update für Exchange 2016 und 2019"
navTitle: "Exchange SU 10/2026"
description: "Das Oktober-SU schliesst sechs Schwachstellen, darunter eine kritische Remote-Code-Execution über eine präparierte E-Mail, und behebt den .ics-Fehler sowie den WordBreaker-Deadlock. Für Exchange 2016 und 2019 ist es das letzte Update, auch im ESU-Programm."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 Min. Lesezeit"
themen:
  - "exchange-updates"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-security-updates-oktober-2026"
translationId: article-173db9431239a95f
url: https://rafaelpfister.ch/blog/exchange-security-updates-oktober-2026
---

# Exchange-Sicherheitsupdates vom Oktober 2026: sechs Schwachstellen, letztes Update für Exchange 2016 und 2019

Microsoft hat am 13. Oktober 2026 Sicherheitsupdates (SUs) für Exchange Server SE, Exchange 2019 und Exchange 2016 veröffentlicht. Laut den KB-Artikeln schliessen sie sechs Schwachstellen. Keine davon war vorab öffentlich bekannt, keine wird laut Security Update Guide aktiv ausgenutzt. Zwei stuft Microsoft aber als «Exploitation More Likely» ein, darunter eine als *Critical* bewertete Remote-Code-Execution, die ein nicht authentifizierter Angreifer mit einer präparierten E-Mail auslösen kann. Besonders ist dieser Monat aus zwei Gründen: Das SU behebt die beiden offenen Known Issues der September-Updates (HTTP 500 bei veröffentlichten Kalendern und den ContentEngine-Deadlock in Exchange SE), und für Exchange 2016 und 2019 ist es laut Microsoft das letzte Sicherheitsupdate überhaupt, auch für Kunden im Period-2-ESU-Programm.

## Für welche Exchange-Versionen das Update verfügbar ist

Die SUs vom 13. Oktober 2026 stehen für folgende Versionen bereit:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, Build 15.2.2562.56; öffentlich verfügbar.
- **Exchange Server 2019 CU15**: KB5128896, Build 15.2.1748.55; nur über das **Period-2-ESU-Programm**.
- **Exchange Server 2019 CU14**: KB5128897, Build 15.2.1544.50; nur über Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5128898, Build 15.1.2507.77; nur über Period 2 ESU.

Den eigenen Stand können Sie mit der Übersicht der [Exchange-Buildnummern](/tools/exchange-builds) abgleichen.

Exchange 2016 und 2019 sind out of support. Laut KB-Artikeln erhalten Organisationen im Period-2-ESU-Programm Sicherheitsupdates bis Ende Oktober 2026; alle anderen sollen auf Exchange SE migrieren. Exchange Online ist laut Ankündigung bereits geschützt; in Hybrid-Umgebungen braucht trotzdem jeder Exchange-Server das SU, auch reine Management-Server und Maschinen mit den Exchange Management Tools. Was das ESU-Ende konkret bedeutet, steht im Abschnitt [Letztes Update für Exchange 2016 und 2019](#letztes-update-für-exchange-2016-und-2019).

## Die Schwachstellen im Überblick

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

Alle vier KB-Artikel führen dieselben sechs CVEs. Im Security Update Guide fehlt bei CVE-2026-87692 jedoch Exchange 2019 CU15; dort sind nur Exchange SE, 2019 CU14 und 2016 CU23 als betroffen aufgeführt (Stand 14. Oktober 2026). Frankys Web liest das so, dass 2019 CU15 von dieser Lücke nicht betroffen ist. Für die Installation spielt der Unterschied keine Rolle, da das SU auf CU15 ohnehin alle übrigen Lücken schliesst.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** ist die einzige Lücke des Monats, die Microsoft als *Critical* einstuft (Remote Code Execution, CVSS 8.1). Die Ursache ist ein stack-basierter Pufferüberlauf. Laut FAQ im Security Update Guide kann ein nicht authentifizierter Angreifer eine präparierte E-Mail an einen betroffenen Exchange-Server senden und damit einen internen Konvertierungsprozess auslösen; eine Benutzerinteraktion ist nicht nötig. Die Angriffskomplexität ist laut CVSS-Vektor hoch (AC:H); Microsoft schreibt dazu, dass eine erfolgreiche Ausnutzung nicht garantiert ist und ein tiefes Verständnis der Zielumgebung erfordert. Trotzdem bewertet Microsoft die Ausnutzung als «Exploitation More Likely». Ein Konto in der Organisation braucht der Angreifer nicht, es genügt, dass der Server Mails annimmt.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** hat mit CVSS 9.3 den höchsten Wert und ist ebenfalls als «Exploitation More Likely» eingestuft. Es handelt sich um ein Cross-Site-Scripting: Ein nicht authentifizierter Angreifer verschickt einen präparierten Link; öffnet ein Administrator ihn, kann der Angreifer Aktionen mit dessen Rechten ausführen und so auf Daten der Organisation zugreifen oder sie ändern. Laut FAQ setzt das eine Benutzerinteraktion voraus, typischerweise über Social Engineering. Der Schweregrad ist *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) setzt nur ein Konto mit geringen Rechten voraus. Laut FAQ kann ein Angreifer einen gemeinsam genutzten Cache des Exchange Control Panel manipulieren und dadurch ein schädliches Mail-Add-in im Postfach eines anderen Benutzers installieren lassen, ohne dass dieser etwas tun muss. Denselben Angriffsweg beschreibt Microsoft für [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), dort als Folge einer unzureichenden Prüfung kryptografischer Signaturen und mit hoher Angriffskomplexität.

Die beiden übrigen Lücken, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (Deserialisierung nicht vertrauenswürdiger Daten) und [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), sind lokale Rechteausweitungen mit CVSS 7.8: Ein angemeldeter Angreifer mit geringen Rechten kann laut Microsoft SYSTEM-Rechte erlangen. Beide sind als «Exploitation Less Likely» eingestuft, ebenso CVE-2026-77910 und CVE-2026-87696.

Zu **CVE-2026-96940** aus der September-v2: Die Oktober-Ankündigung erwähnt die v2 nicht. Laut FAQ der Ankündigung sind SUs kumulativ; wer auf einem unterstützten CU ist, installiert nur das neueste SU. Frankys Web bestätigt, dass das Oktober-SU den Fix aus der v2 enthält. Server, auf denen noch das erste September-SU vom 8. September läuft, können also direkt das Oktober-SU installieren; Details zur v2 stehen im [Artikel zur September-v2](/blog/exchange-security-updates-september-2026-v2).

## Letztes Update für Exchange 2016 und 2019

Die Ankündigung ist in diesem Punkt eindeutig: Oktober 2026 ist der letzte Monat, in dem Kunden im Period-2-ESU-Programm neue Sicherheitsupdates für Exchange 2016 und 2019 erhalten. Ab dem nächsten Release erscheinen Updates nur noch für Exchange SE. Künftige Ankündigungen und CVE-Meldungen werden 2016 und 2019 nicht mehr nennen, und Microsoft schreibt ausdrücklich: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» Dass eine CVE diese Versionen nicht erwähnt, bedeutet also nicht, dass sie nicht betroffen sind; Microsoft betont in der Ankündigung, dass die alten Versionen dadurch nicht sicher werden. Frankys Web weist zusätzlich darauf hin, dass Exchange SE und 2019 CU15 technisch sehr ähnlich sind, künftige SE-Lücken also mit hoher Wahrscheinlichkeit auch 2019 betreffen.

Period 2 ESU lässt sich laut Ankündigung noch für begrenzte Zeit kaufen. Das bringt eine Organisation aber nur auf den Stand des Oktober-2026-Updates, weitere Updates folgen nicht. Wer ein ESU bereits erworben hat und Fragen zum Zugriff hat, kann sich laut KB-Artikeln an `ExchangeandSfBServerESUInquiry@service.microsoft.com` wenden.

Für den Betrieb heisst das:

1. **Exchange 2016 und 2019 mit ESU**: Installieren Sie das Oktober-SU. Danach bleibt der Stand eingefroren, und jede später bekannt gewordene Lücke bleibt offen.

2. **Exchange 2016 und 2019 ohne ESU**: Diese Server haben seit Mai 2026 keine Sicherheitsupdates mehr erhalten. Microsoft empfiehlt die Migration auf Exchange SE so bald wie möglich.

3. **Hybrid-Umgebungen**: Zusätzlich zum fehlenden Patch drosselt und blockiert Exchange Online den Mailfluss von veralteten On-Premises-Servern; die Regeln stehen im [Artikel zum Transport-Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Behobene Probleme und was Sie danach tun müssen

**Veröffentlichte Kalender (.ics) liefern HTTP 500 an Kalender-Apps** (alle Versionen). Das seit dem August-SU bestehende Problem ist mit dem Oktober-SU behoben; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) nennt als Lösung jetzt die Installation der vier Oktober-Updates oder neuerer. Den bisherigen URL-Rewrite-Workaround auf der Site «Exchange Back End» beschreibt der Support-Artikel nicht mehr, und eine Anleitung zum Entfernen der Regel gibt Microsoft nicht (Stand 14. Oktober 2026). Haben Sie die Regel eingerichtet, dokumentieren Sie, auf welchen Servern sie aktiv ist, und testen Sie ein Kalenderabonnement, bevor Sie sie entfernen.

**ContentEngine-Deadlock wegen fehlender koreanischer WordBreaker-Dateien** (nur Exchange SE). Laut [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) ist das Problem mit KB5128895 oder einem neueren Update behoben, und nach der Installation des Oktober-Updates ist «no additional action» nötig. Das gilt auch für Server, auf denen Sie den manuellen Workaround aus dem [September-Artikel](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock) angewendet haben: Einen Rückbau der kopierten Regeldateien verlangt Microsoft nicht. Server ohne Workaround müssen die Dateien nicht mehr manuell ergänzen.

**Frei/Gebucht für delegierte Postfächer in Hybrid-Umgebungen mit reiner Graph-API-Konfiguration** (nur Exchange SE). Dieses Problem behebt nicht das Oktober-SU: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) meldet jetzt «The issue has been fixed in the service». Der Widerspruch aus dem [v2-Artikel](/blog/exchange-security-updates-september-2026-v2) ist damit aufgelöst. Haben Sie den Workaround gesetzt, nehmen Sie ihn laut KB5127092 zurück:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Zeigt den aktuellen Wert des Overrides; nur bei `Enabled=False` ist der nächste Befehl nötig. |
| `-Parameters "Enabled=True"` | Schaltet die Weiterleitung der Verfügbarkeitsabfragen über Microsoft Graph wieder ein. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Richtet den Diagnoseaufruf an den Topologiedienst. |
| `-Component VariantConfiguration -Argument Refresh` | Lädt die Variant Configuration neu, damit die Änderung sofort greift. |

</details>

Prüfen Sie danach die Frei/Gebucht-Abfrage eines delegierten Exchange-Online-Postfachs aus Outlook oder OWA.

## Bekannte Probleme

Weder die Ankündigung noch die vier KB-Artikel nennen Known Issues für das Oktober-SU (Stand 14. Oktober 2026). Am 15. Oktober 2026 hat Microsoft die Ankündigung ergänzt: Die Windows-Server-Updates vom Oktober 2026 können dazu führen, dass Exchange-Server nach rund einem Tag mit `MapiExceptionMdbOffline` (ec=1142) ausfallen, auch ohne installiertes Exchange-SU. Symptome und Vorgehen stehen im [Artikel zu den Exchange-Ausfällen nach den Windows-Updates](/blog/exchange-windows-updates-oktober-2026). Die drei offenen Probleme aus den September-Updates sind damit erledigt: zwei durch das Oktober-SU, eines durch eine Korrektur in Exchange Online (siehe oben). Das Wrapper-Problem in freigegebenen Postfächern war bereits mit dem September-SU behoben.

## Installation und Nachbereitung

Microsoft nennt in der Ankündigung den bekannten Ablauf: mit dem [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) inventarisieren, bei veraltetem CU-Stand mit dem [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) den Pfad ermitteln, das SU installieren, den Server neu starten und prüfen, ob alle Exchange-Dienste laufen. Bei Fehlern verweist Microsoft auf das [SetupAssist-Skript](https://aka.ms/ExSetupAssist). Nach der Installation zeigt der Health Checker laut KB5128895, ob das SU korrekt installiert ist und weitere Schritte nötig sind. Der Security Update Guide führt für die Oktober-Builds keinen erforderlichen Neustart; die Ankündigung empfiehlt ihn nach dem Setup trotzdem.

Die Installationsdatei für Exchange SE heisst laut KB5128895 `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), für Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, für CU14 `Exchange2019-KB5128897-x64-en.exe` und für Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. Die Pakete sind CU-spezifisch.

Für Hybrid-Umgebungen gilt laut FAQ: Wird nach der Installation eines SUs das Auth-Zertifikat gewechselt, ist der Hybrid Configuration Wizard erneut auszuführen. Auf Windows Server 2025 erscheinen installierte Exchange-SUs nicht in der Systemsteuerung; Microsoft rät davon ab, SUs zu deinstallieren.

Aus den Vormonaten bleiben diese Nacharbeiten offen, falls noch nicht erledigt: den Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` entfernen (siehe [September-Artikel](/blog/exchange-security-updates-september-2026)) und prüfen, ob die CVE-2026-42897-Mitigation (M2.1.0) noch aktiv ist (siehe [Artikel zum Juli-SU](/blog/exchange-security-updates-juli-2026)).

## Empfohlenes Vorgehen

Installieren Sie das Oktober-SU zeitnah auf allen Exchange-Servern und Management-Tools-Maschinen: Die kritische Remote-Code-Execution CVE-2026-70571 lässt sich ohne Konto über eine E-Mail auslösen, und Microsoft hält ihre Ausnutzung ebenso für wahrscheinlicher wie die des Cross-Site-Scriptings CVE-2026-70279. Server mit dem ersten September-SU erhalten damit auch den Fix für CVE-2026-96940. Nehmen Sie danach den Frei/Gebucht-Override zurück, falls gesetzt, und prüfen Sie veröffentlichte Kalender. Für Exchange 2016 und 2019 ist dies das letzte Update; planen Sie die Migration auf Exchange SE jetzt, denn ab November erhalten diese Server keine Sicherheitsupdates mehr.

## Quellen

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Ankündigung vom 13. Oktober 2026 mit unterstützten Versionen, behobenen Problemen, dem Ende der Updates für Exchange 2016/2019 und FAQ (eingesehen über den RSS-Feed des Exchange Team Blogs; keine Kommentare des Exchange-Teams, Stand 14. Oktober 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): CVE-Liste, behobene Probleme, Installationsdatei und Hinweis auf den Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): KB-Artikel für Exchange 2019 CU15 mit ESU-Hinweis.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): KB-Artikel für Exchange 2019 CU14 mit ESU-Kontaktadresse.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): KB-Artikel für Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, Schweregrad, Ausnutzungseinschätzung, FAQ und betroffene Builds der sechs Oktober-CVEs sowie von CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummern der Oktober-SUs und der Vorgänger-Updates.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Fix mit den Oktober-Updates.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Fix mit KB5128895, keine weiteren Schritte nötig.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Korrektur im Dienst und Befehle zum Zurücksetzen des SettingOverrides.

11. [Neue Sicherheitsupdates für Exchange Server (Oktober 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): Deutschsprachige Aufschlüsselung der sechs CVEs, Hinweis auf den enthaltenen v2-Fix und Einordnung des ESU-Endes.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisierung von CU/SU-Ständen vor und nach der Installation.
