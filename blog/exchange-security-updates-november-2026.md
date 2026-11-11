---
title: "Exchange-Sicherheitsupdates vom November 2026: neun Schwachstellen, erstes Update nur noch für Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "Das November-SU schliesst neun Schwachstellen in Exchange Server SE, darunter eine kritische Remote-Code-Execution, für die ein Konto mit geringen Rechten genügt. Exchange 2016 und 2019 erhalten kein Update mehr. Die Exchange-Ausfälle nach den Windows-Updates vom Oktober behebt das Windows-Update vom November."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 Min. Lesezeit"
themen:
  - "exchange-updates"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "exchange-security-updates-november-2026"
translationId: article-cfb739155e545cc3
url: https://rafaelpfister.ch/blog/exchange-security-updates-november-2026
---

# Exchange-Sicherheitsupdates vom November 2026: neun Schwachstellen, erstes Update nur noch für Exchange SE

Microsoft hat am 10. November 2026 ein Sicherheitsupdate (SU) für Exchange Server SE veröffentlicht. Laut KB-Artikel schliesst es neun Schwachstellen. Keine davon war vorab öffentlich bekannt, keine wird laut Security Update Guide aktiv ausgenutzt. Eine Remote-Code-Execution stuft Microsoft als *Critical* und als «Exploitation More Likely» ein; für sie genügt ein Konto mit geringen Rechten. Besonders ist dieser Monat aus zwei Gründen: Es ist das erste SU, das nur noch für Exchange SE erscheint, und die Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober sind mit den Windows-Updates vom November behoben.

## Für welche Exchange-Versionen das Update verfügbar ist

Das SU vom 10. November 2026 steht nur für eine Version bereit:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, Build 15.2.2562.57; öffentlich verfügbar.

Den eigenen Stand können Sie mit der Übersicht der [Exchange-Buildnummern](/tools/exchange-builds) abgleichen.

Für Exchange 2016 und 2019 gibt es kein Update. Das [Oktober-SU](/blog/exchange-security-updates-oktober-2026) war das letzte, auch für Kunden im Period-2-ESU-Programm, und laut Ankündigung ist dieses Programm beendet. Exchange Online ist laut Microsoft bereits geschützt; in Hybrid-Umgebungen braucht trotzdem jeder Exchange-Server das SU, auch reine Management-Server und Maschinen mit den Exchange Management Tools.

## Die Schwachstellen im Überblick

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Remote Code Execution | 8.8 |
| CVE-2026-118149 | Remote Code Execution | 8.8 |
| CVE-2026-118150 | Elevation of Privilege | 8.8 |
| CVE-2026-118154 | Elevation of Privilege | 8.8 |
| CVE-2026-118153 | Denial of Service | 7.5 |
| CVE-2026-118151 | Elevation of Privilege | 7.2 |
| CVE-2026-118157 | Elevation of Privilege | 7.2 |
| CVE-2026-118208 | Spoofing | 7.1 |
| CVE-2026-118152 | Spoofing | 6.5 |

Alle neun Lücken sind über das Netzwerk erreichbar (AV:N) und haben laut CVSS-Vektor eine niedrige Angriffskomplexität (AC:L). Der Security Update Guide führt als betroffenes Produkt nur Exchange Server SE RTM.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207)** ist die einzige Lücke des Monats, die Microsoft als *Critical* einstuft (Remote Code Execution, CVSS 8.8), und die einzige mit der Einschätzung «Exploitation More Likely». Laut FAQ im Security Update Guide kann ein authentifizierter Angreifer mit geringen Rechten eine präparierte Anfrage an einen betroffenen Server senden und damit Code auf dem Server ausführen; eine Benutzerinteraktion ist nicht nötig. Als Ursache nennt Microsoft eine unzureichende Eingabeprüfung (CWE-706). Für den Angriff reicht also ein gewöhnliches Postfachkonto, zum Beispiel eines mit einem per Phishing erbeuteten Kennwort.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149)** ist die zweite Remote-Code-Execution (CVSS 8.8, *Important*). Auch hier sendet ein authentifizierter Angreifer laut FAQ eine präparierte Anfrage an einen betroffenen Dienst und kann danach Code auf dem Zielsystem ausführen; die Ursache ist eine Code-Injection (CWE-94). Microsoft bewertet die Ausnutzung als «Exploitation Unlikely».

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153)** (Denial of Service, CVSS 7.5) setzt als einzige Lücke weder ein Konto noch eine Benutzerinteraktion voraus. Laut Beschreibung kann ein nicht authentifizierter Angreifer den Dienst über das Netzwerk ausser Betrieb setzen; die Ursache ist eine ineffiziente algorithmische Komplexität (CWE-407). Betroffen ist laut Vektor nur die Verfügbarkeit, nicht die Vertraulichkeit oder Integrität.

Die vier Rechteausweitungen unterscheiden sich in der Voraussetzung: Für [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) und [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (beide CVSS 8.8) genügt ein Konto mit geringen Rechten, für [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) und [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (beide CVSS 7.2) braucht der Angreifer bereits hohe Rechte. Bei CVE-2026-118151 kann er laut FAQ SYSTEM-Rechte erlangen.

Von den beiden Spoofing-Lücken verlangt [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) ein Konto mit geringen Rechten. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) ist ein Cross-Site-Request-Forgery: Der Angreifer braucht kein Konto, ein Benutzer muss aber mitwirken (UI:R). Alle Lücken ausser CVE-2026-118207 und CVE-2026-118149 sind als «Exploitation Less Likely» eingestuft.

## Exchange 2016 und 2019: kein Update, keine Prüfung mehr

Die Ankündigung nennt Exchange 2016 und 2019 nur noch in einem Hinweis: Diese Versionen werden nicht mehr auf Schwachstellen getestet, und es erscheinen keine Updates mehr. Microsoft schreibt, man solle sie als dauerhaft verwundbar betrachten («permanently vulnerable») und die Organisation sofort aktualisieren. Dass die neun November-CVEs nur Exchange SE aufführen, sagt deshalb nichts darüber aus, ob die älteren Versionen betroffen sind. Frankys Web weist darauf hin, dass Exchange SE und 2019 CU15 technisch nahe beieinander liegen und zumindest ein Teil der Lücken vermutlich auch in Exchange 2019 vorhanden ist.

Für den Betrieb heisst das:

1. **Exchange 2019 CU15 und älter, Exchange 2016**: Der Stand bleibt auf dem Oktober-SU (mit ESU) oder älter eingefroren. Die neun Lücken vom November bleiben auf diesen Servern offen, falls sie dort vorhanden sind, und Microsoft prüft das nicht mehr.

2. **Migration**: Microsoft empfiehlt den sofortigen Umstieg auf Exchange SE.

3. **Hybrid-Umgebungen**: Exchange Online drosselt und blockiert den Mailfluss von veralteten On-Premises-Servern; die Regeln stehen im [Artikel zum Transport-Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Behobene Probleme

**Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober** (alle Versionen). Das Problem mit `MapiExceptionMdbOffline` (ec=1142) behebt nicht das Exchange-SU: Laut Ankündigung sind die Windows-Server-Updates vom November 2026 zu installieren. Die Release Notes dieser Updates führen den Fix unter «[Exchange Server]» und nennen das Problem als gelöst mit Updates ab dem 10. November 2026: KB5131080 für Windows Server 2025, KB5131090 für Windows Server 2022, KB5131085 für Windows Server 2019 und KB5131291 für Windows Server 2016. Die Reihenfolge, in der Sie das Exchange-SU und das Windows-Update installieren, spielt laut Ankündigung keine Rolle. Haben Sie den Registry-Wert `EpmapSavedContextCap` aus dem [Artikel zu den Exchange-Ausfällen](/blog/exchange-windows-updates-oktober-2026) gesetzt, können Sie ihn stehen lassen; nach der Installation der November-Updates für Windows Server ist er laut Microsoft nicht mehr nötig.

**Falsche Frei/Gebucht-Anzeige über Microsoft Graph in Hybrid-Umgebungen** (nur Exchange SE). Laut [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867) sahen Benutzer mit On-Premises-Postfach im klassischen Outlook für Exchange-Online-Postfächer keine korrekten Frei/Gebucht-Informationen, wenn die Graph API aktiviert war; angezeigt wurden nur Zeitblöcke. Mit Exchange Web Services (EWS) statt der Graph API stimmte die Anzeige. Das Problem ist mit KB5133453 oder einem neueren Update behoben. Eine Nacharbeit nennt der Support-Artikel nicht. Haben Sie wegen dieses Fehlers auf EWS zurückgestellt, prüfen Sie nach der Installation die Frei/Gebucht-Abfrage eines Exchange-Online-Postfachs, bevor Sie die Graph API wieder aktivieren.

## Bekannte Probleme

Weder die Ankündigung noch KB5133453 nennen Known Issues für das November-SU (Stand 11. November 2026). Die Ankündigung hat keine Kommentare des Exchange-Teams, und Microsoft weist darauf hin, dass die Dokumentation zum Zeitpunkt der Veröffentlichung noch unvollständig sein kann. Aus den Vormonaten ist kein Known Issue mehr offen: Die Probleme der September-Updates sind seit dem Oktober-SU erledigt, und das Windows-Problem vom Oktober ist mit den November-Updates für Windows Server behoben (siehe oben).

## Installation und Nachbereitung

Microsoft nennt in der Ankündigung den bekannten Ablauf: mit dem [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) inventarisieren, welche Server Updates oder manuelle Schritte brauchen, das SU installieren, den Server neu starten und prüfen, ob alle Exchange-Dienste laufen. Sind Dienste nach dem Setup deaktiviert, wurde die Installation unterbrochen. Bei Fehlern während oder nach der Installation verweist Microsoft auf das [SetupAssist-Skript](https://aka.ms/ExSetupAssist). Nach der Installation zeigt der Health Checker laut KB5133453, ob das SU korrekt installiert ist und weitere Schritte nötig sind.

Die Installationsdatei heisst laut KB5133453 `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: «SU12»). Das SU ersetzt das Oktober-SU KB5128895. SUs sind kumulativ: Wer ein älteres SU auf Exchange SE RTM installiert hat, installiert direkt das November-SU.

Für Hybrid-Umgebungen gilt laut FAQ: Wird nach der Installation eines SUs das Auth-Zertifikat gewechselt, ist der Hybrid Configuration Wizard erneut auszuführen. Auf Windows Server 2025 erscheinen installierte Exchange-SUs nicht in der Systemsteuerung; Microsoft rät davon ab, SUs zu deinstallieren.

Aus den Vormonaten bleiben diese Nacharbeiten offen, falls noch nicht erledigt: den Frei/Gebucht-Override `EnableRouteThroughMSGraphFeature` zurücknehmen (siehe [Oktober-Artikel](/blog/exchange-security-updates-oktober-2026)), den Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` entfernen (siehe [September-Artikel](/blog/exchange-security-updates-september-2026)) und prüfen, ob die CVE-2026-42897-Mitigation (M2.1.0) noch aktiv ist (siehe [Artikel zum Juli-SU](/blog/exchange-security-updates-juli-2026)).

## Empfohlenes Vorgehen

Installieren Sie das November-SU zeitnah auf allen Servern mit Exchange SE und auf den Maschinen mit den Management Tools: Für die kritische Remote-Code-Execution CVE-2026-118207 genügt ein Konto mit geringen Rechten, und Microsoft hält ihre Ausnutzung für wahrscheinlicher. Installieren Sie im selben Wartungsfenster die November-Updates für Windows Server, damit die Ursache der Oktober-Ausfälle behoben ist. Für Exchange 2016 und 2019 gibt es keinen Patch mehr; diese Server sind laut Microsoft als dauerhaft verwundbar zu betrachten und sollten sofort durch Exchange SE ersetzt werden.

## Quellen

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): Ankündigung vom 10. November 2026 mit unterstützter Version, Hinweis zu Exchange 2016/2019, dem behobenen Windows-Problem und FAQ (keine Kommentare des Exchange-Teams, Stand 11. November 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): CVE-Liste, behobenes Problem, Installationsdatei, ersetztes Update und Hinweis auf den Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, Schweregrad, Ausnutzungseinschätzung, FAQ und betroffenes Produkt der neun November-CVEs.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummer des November-SUs.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): Symptome und Fix mit KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): Beitrag des Exchange-Teams zum Windows-Problem vom Oktober, auf den die Ankündigung verweist.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): Release Notes für Windows Server 2025 mit dem Exchange-Fix und dem als gelöst markierten Known Issue.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): Release Notes für Windows Server 2022 mit demselben Fix.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): Release Notes für Windows Server 2019 mit demselben Fix.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): Release Notes für Windows Server 2016 mit demselben Fix.

11. [Neue Sicherheitsupdates für Exchange Server (November 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): Deutschsprachige Aufschlüsselung der neun CVEs und Einordnung für Exchange 2016 und 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisierung von CU/SU-Ständen vor und nach der Installation.
