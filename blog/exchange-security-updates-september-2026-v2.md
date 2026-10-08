---
title: "Exchange-Sicherheitsupdate September 2026 v2: vorgezogener Fix für CVE-2026-96940"
navTitle: "Exchange SU 09/2026 v2"
description: "Am 2. Oktober 2026 hat Microsoft eine Version 2 der September-SUs für Exchange SE, 2019 und 2016 veröffentlicht. Sie schliesst zusätzlich CVE-2026-96940, eine Elevation-of-Privilege-Lücke mit CVSS 8.8 und der Einschätzung «Exploitation More Likely». Microsoft empfiehlt, v2 so bald wie möglich zu installieren, auch auf Servern mit dem ersten September-SU."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "5 Min. Lesezeit"
themen:
  - "exchange-updates"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "exchange-security-updates-september-2026-v2"
translationId: article-3ae7bbe7175754d9
url: https://rafaelpfister.ch/blog/exchange-security-updates-september-2026-v2
---

# Exchange-Sicherheitsupdate September 2026 v2: vorgezogener Fix für CVE-2026-96940

Microsoft hat am 2. Oktober 2026 eine **Version 2 (v2)** der Sicherheitsupdates (SUs) vom September für Exchange Server SE, Exchange 2019 und Exchange 2016 veröffentlicht. Laut Ankündigung besteht der einzige Unterschied zur ersten Fassung vom 8. September darin, dass v2 zusätzlich **CVE-2026-96940** schliesst. Die Lücke ist weder öffentlich bekannt noch wird sie ausgenutzt; Microsoft hat sie intern gefunden. Ungewöhnlich ist zweierlei: Microsoft hat dieses Update nach eigener Aussage früher als geplant veröffentlicht, und der Security Update Guide stuft die Ausnutzung als **«Exploitation More Likely»** ein. Die neun Schwachstellen der ersten Fassung, das behobene Wrapper-Problem und die Known Issues sind im [Artikel zum September-SU](/blog/exchange-security-updates-september-2026) beschrieben; dieser Artikel behandelt, was mit v2 neu ist.

## Für welche Exchange-Versionen das Update verfügbar ist

Die v2-SUs vom 2. Oktober 2026 stehen für folgende Versionen bereit:

- **Exchange Server Subscription Edition (SE) RTM**: KB5129955, Build 15.2.2562.53; öffentlich verfügbar.
- **Exchange Server 2019 CU15**: KB5129956, Build 15.2.1748.53; nur über das **Period-2-ESU-Programm**.
- **Exchange Server 2019 CU14**: KB5129957, Build 15.2.1544.48; nur über Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5129958, Build 15.1.2507.75; nur über Period 2 ESU.

Die Updates sind CU-spezifisch: Das Paket für CU15 lässt sich nicht auf CU14 installieren. Exchange 2016 und 2019 sind out of support. Laut FAQ der Ankündigung erhalten nur Organisationen im Period-2-ESU-Programm, das von Mai bis Oktober 2026 gilt, nach Mai 2026 veröffentlichte Updates für diese Versionen; allen anderen empfiehlt Microsoft den Wechsel auf Exchange SE so bald wie möglich. Für Hybrid-Umgebungen mit veralteten Servern kommt das [Transport-Enforcement von Exchange Online](/blog/exchange-online-transport-enforcement-hybrid-server) hinzu.

Ob Ihre Server bereits auf dem v2-Stand sind, können Sie mit der Übersicht der [Exchange-Buildnummern](/tools/exchange-builds) prüfen.

## Die Schwachstelle im Überblick

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-96940 | Elevation of Privilege | 8.8 |

**[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)** beschreibt Microsoft als schwache Autorisierung in Exchange Server, über die ein authentifizierter Angreifer über das Netzwerk seine Rechte ausweiten kann. Der CVSS-Vektor nennt geringe Anforderungen: Angriff über das Netzwerk, geringe Komplexität, ein Konto mit niedrigen Rechten, keine Benutzerinteraktion. Laut FAQ im Security Update Guide erhält ein erfolgreicher Angreifer unberechtigten Zugriff auf die Postfächer anderer Benutzer derselben Organisation und kann deren Mails samt Anhängen lesen; der Zugriff bleibt auf die eigene Organisation beschränkt. Microsoft bewertet die Lücke als *Important*, der Temporal Score liegt bei 7.7.

Die Einschätzung «Exploitation More Likely» ist der wichtigste Unterschied zu den neun CVEs der ersten Fassung, die Microsoft alle mit «Exploitation Less Likely» bewertet hat. Ausgangspunkt eines Angriffs ist ein beliebiges gültiges Benutzerkonto, etwa nach einem erfolgreichen Phishing. Exchange Online hat Microsoft serverseitig korrigiert; dort ist nichts zu tun. Eine Mitigation oder einen Workaround für Server ohne v2 nennt Microsoft nicht (Stand 7. Oktober 2026).

Die übrigen CVEs in den KB-Artikeln der v2 sind dieselben wie in der ersten Fassung: CVE-2026-55007, CVE-2026-69355, CVE-2026-69356, CVE-2026-69361, CVE-2026-69375, CVE-2026-69378, CVE-2026-69382 und CVE-2026-69641. Wie schon bei den September-KBs fehlt dort CVE-2026-69380, die laut Security Update Guide ebenfalls mit dem September-SU geschlossen wird.

## Warum es eine v2 gibt und was Sie tun müssen

In der FAQ der Ankündigung beantwortet Microsoft die Frage nach der ungewöhnlichen Reihenfolge so: Das Update mit CVE-2026-96940 sei vor dem geplanten Termin veröffentlicht worden, und Microsoft empfiehlt, die Deployment-Hinweise zu prüfen und das Update bei der frühestmöglichen Gelegenheit zu installieren. Warum der Termin vorgezogen wurde, schreibt Microsoft nicht.

Daraus ergibt sich für die Praxis:

1. **Server mit dem SU vom 8. September** (Builds 15.2.2562.49, 15.2.1748.51, 15.2.1544.46, 15.1.2507.73) sind gegen CVE-2026-96940 nicht geschützt. Der Security Update Guide führt für diese CVE ausschliesslich die vier v2-Builds als Fix. Diese Server brauchen v2.

2. **Server mit einem älteren Stand** können direkt v2 installieren. Laut FAQ sind SUs kumulativ; wer auf einem unterstützten CU ist, installiert nur das neueste SU.

3. **Maschinen mit den Exchange Management Tools** und reine Management-Server erhalten v2 ebenfalls; das empfiehlt Microsoft für alle SUs, auch in Hybrid-Umgebungen.

Die Installationsdatei für Exchange SE heisst laut KB5129955 `ExchangeSubscriptionEdition-KB5129955-x64-en.exe`; sie ist über den Microsoft Update Catalog und das Download Center erhältlich, dort unter der Bezeichnung «SU10V2».

## Bekannte Probleme

Die Known Issues der ersten Fassung gelten auch für v2. Stand 7. Oktober 2026:

**Veröffentlichte Kalender (.ics) liefern HTTP 500 an Kalender-Apps** (alle Versionen). Das Problem besteht seit dem August-SU und steht in allen vier v2-KBs weiter als Known Issue. Microsoft untersucht es noch und beschreibt als vorübergehenden Workaround eine URL-Rewrite-Regel auf der Site «Exchange Back End»; die Schritte stehen im [Support-Artikel KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) und im [Artikel zum September-SU](/blog/exchange-security-updates-september-2026). Die Regel ist auf jedem Server einzurichten und laut Microsoft nach künftigen Updates erneut zu prüfen.

**ContentEngine-Deadlock wegen fehlender koreanischer WordBreaker-Dateien** (nur Exchange SE). Betroffen sind laut [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) ausdrücklich beide Fassungen, Build 15.2.2562.49 und der v2-Build 15.2.2562.53. Die v2 behebt das Problem also nicht. Symptome sind fehlende Suchergebnisse, verzögerte Mailzustellung und hängende oder getrennte Outlook- bzw. MAPI-Clients; laut Ankündigung betrifft es Organisationen mit Mails in koreanischer Sprache. Der manuelle Workaround (zwei Regeldateien aus SQL Server 2025 Express RTM, Prüfung per SHA256-Hash) steht im Support-Artikel; Microsoft untersucht das Problem weiter.

**Frei/Gebucht für delegierte Postfächer in Hybrid-Umgebungen mit reiner Graph-API-Konfiguration** (nur Exchange SE). Hier widersprechen sich die Quellen: Die v2-Ankündigung führt das Problem unter den behobenen Problemen, KB5129955 listet es weiterhin als Known Issue, und der [Support-Artikel KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) meldet den Stand «Microsoft is investigating this issue» ohne Hinweis auf einen Fix. Haben Sie den dort beschriebenen SettingOverride `EnableRouteThroughMSGraphFeature` als Workaround gesetzt, nehmen Sie ihn nach der Installation von v2 nur nach einem Test zurück, solange Microsoft keine Anleitung dazu veröffentlicht.

## Installation und Nachbereitung

Microsoft nennt in der Ankündigung den bekannten Ablauf: mit dem [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) inventarisieren, bei veraltetem CU-Stand mit dem [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) den Pfad ermitteln, nach dem Setup den Server neu starten und prüfen, ob alle Exchange-Dienste gestartet sind. Bei Fehlern während oder nach der Installation verweist Microsoft auf das [SetupAssist-Skript](https://aka.ms/ExSetupAssist). Der Security Update Guide führt für die v2-Builds bei CVE-2026-96940 keinen erforderlichen Neustart; die Ankündigung empfiehlt den Neustart nach dem Setup trotzdem.

Für Hybrid-Umgebungen gilt laut FAQ: Wird nach der Installation eines SUs das Auth-Zertifikat gewechselt, ist der Hybrid Configuration Wizard erneut auszuführen. Auf Windows Server 2025 erscheinen installierte Exchange-SUs nicht in der Systemsteuerung; Microsoft rät generell davon ab, SUs zu deinstallieren, und verweist für diesen Fall auf einen eigenen Support-Artikel.

Aus den Vormonaten bleiben diese Nacharbeiten offen, falls noch nicht erledigt: den Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` entfernen (siehe [September-Artikel](/blog/exchange-security-updates-september-2026)) und prüfen, ob die CVE-2026-42897-Mitigation (M2.1.0) noch aktiv ist (siehe [Artikel zum Juli-SU](/blog/exchange-security-updates-juli-2026)).

## Empfohlenes Vorgehen

Installieren Sie v2 zeitnah auf allen Exchange-Servern und Management-Tools-Maschinen, auch dort, wo das September-SU bereits läuft: Nur die v2-Builds schliessen CVE-2026-96940, ein einzelnes kompromittiertes Benutzerkonto genügt für den Zugriff auf fremde Postfächer, und Microsoft hält eine Ausnutzung für wahrscheinlicher als bei allen anderen September-CVEs. Prüfen Sie anschliessend mit dem Health Checker den Build-Stand, kontrollieren Sie auf Exchange SE das WordBreaker-Problem und behalten Sie bestehende Workarounds bei, bis Microsoft deren Rücknahme dokumentiert. Für Exchange 2016 und 2019 endet das ESU-Programm im Oktober 2026; danach erscheinen für diese Versionen keine Updates mehr.

## Quellen

1.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Ankündigung der v2 vom 2. Oktober 2026 mit dem Unterschied zur ersten Fassung, der FAQ zur vorgezogenen Veröffentlichung, Known Issues, behobenen Problemen und Installationsablauf (eingesehen über den RSS-Feed des Exchange Team Blogs).

2.  [CVE-2026-96940 – Security Update Guide, Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940): Typ, CVSS 8.8/7.7 mit Vektor, Schweregrad, Ausnutzungseinschätzung, FAQ und die vier v2-Builds als Fix.

3.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): CVE-Liste, Known Issues, behobene Probleme und Installationsdatei für Exchange SE.

4.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU15 (KB5129956) – Microsoft Support](https://support.microsoft.com/help/5129956): KB-Artikel für Exchange 2019 CU15 mit ESU-Hinweis.

5.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU14 (KB5129957) – Microsoft Support](https://support.microsoft.com/help/5129957): KB-Artikel für Exchange 2019 CU14.

6.  [Description of version 2 of the security update for Microsoft Exchange Server 2016 CU23 October 2, 2026 (KB5129958) – Microsoft Support](https://support.microsoft.com/help/5129958): KB-Artikel für Exchange 2016 CU23.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummern der September-SUs und der v2 vom 2. Oktober 2026.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Betroffene Versionen, Status und URL-Rewrite-Workaround.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Betroffene SE-Builds einschliesslich v2 und manueller Workaround.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptome, SettingOverride-Workaround und Stand der Untersuchung.

11. [V2 Security Updates Exchange 2016-SE (Sep2026) – EighTwOne](https://eightwone.com/2026/10/03/v2-security-updates-exchange-2016-se-sep2026/): Übersicht der v2-Builds und KBs, Hinweis auf CU-spezifische Pakete.
