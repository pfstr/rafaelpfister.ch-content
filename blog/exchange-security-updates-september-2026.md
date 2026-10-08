---
title: "Exchange-Sicherheitsupdates vom September 2026: neun Schwachstellen, Wrapper-Problem behoben, v2 nachgeschoben"
navTitle: "Exchange SU 09/2026"
description: "Das September-SU schliesst neun Schwachstellen in Exchange SE und 2019 (acht in Exchange 2016), darunter eine Spoofing-Lücke mit CVSS 9.3, und behebt das Wrapper-Problem in Hybrid-Umgebungen. Am 2. Oktober folgte eine v2 mit einer zusätzlichen CVE; dazu kommen drei Known Issues mit Workarounds und ein SettingOverride, der jetzt entfernt werden soll."
date: "2026-10-07"
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
slug: "exchange-security-updates-september-2026"
translationId: article-53db0c02bc33f9bd
url: https://rafaelpfister.ch/blog/exchange-security-updates-september-2026
---

# Exchange-Sicherheitsupdates vom September 2026: neun Schwachstellen, Wrapper-Problem behoben, v2 nachgeschoben

Microsoft hat am 8. September 2026 Sicherheitsupdates (SUs) für Exchange Server veröffentlicht. Sie schliessen in Exchange SE und Exchange 2019 neun Schwachstellen, in Exchange 2016 acht. Keine davon war vorab öffentlich bekannt, keine wird laut Security Update Guide aktiv ausgenutzt, und Microsoft stuft alle als *Important* mit «Exploitation Less Likely» ein. Der höchste CVSS-Wert liegt mit 9.3 aber deutlich über dem des Vormonats. Besonders ist dieser Monat aus drei Gründen: Das SU behebt das seit Juni offene Problem der *Wrapper-Nachrichten* in freigegebenen Postfächern, es bringt drei neue bzw. fortbestehende Known Issues mit sich, und am 2. Oktober hat Microsoft eine **Version 2** nachgeschoben, die eine zusätzliche Schwachstelle schliesst.

## Für welche Exchange-Versionen das Update verfügbar ist

Die SUs vom 8. September 2026 stehen für folgende Versionen bereit:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, Build 15.2.2562.49; öffentlich verfügbar.
- **Exchange Server 2019 CU15**: KB5121609, Build 15.2.1748.51; nur über das **Period-2-ESU-Programm**.
- **Exchange Server 2019 CU14**: KB5121610, Build 15.2.1544.46; nur über Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121611, Build 15.1.2507.73; nur über Period 2 ESU.

Exchange 2016 und 2019 sind out of support. Die SUs von Mai bis Oktober 2026 erhalten laut Microsoft nur Organisationen, die im Period-2-ESU-Programm eingeschrieben sind. Laut den KB-Artikeln reicht diese Berechtigung bis Oktober 2026. Hinzu kommt der Druck aus Exchange Online: Seit der zweiten Septemberwoche drosselt und blockiert Exchange Online den Hybrid-Mailfluss von Servern unter dem Oktober-2025-Stand, Details im [Artikel zum Transport-Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online selbst ist laut Ankündigung bereits geschützt; in Hybrid-Umgebungen braucht trotzdem jeder Exchange-Server das SU, ebenso Maschinen mit den Exchange Management Tools.

Den eigenen Stand können Sie mit der Übersicht der [Exchange-Buildnummern](/tools/exchange-builds) abgleichen.

## Die Schwachstellen im Überblick

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site-Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 betrifft Exchange 2016 nicht; der Security Update Guide führt dafür nur Exchange SE und 2019 CU14/CU15. Ein Detail zur Dokumentation: In den KB-Artikeln der vier September-SUs fehlt CVE-2026-69380 in der CVE-Liste. Der Security Update Guide nennt für diese CVE jedoch genau die vier September-Builds als Fix (Stand 7. Oktober 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** hat mit CVSS 9.3 den höchsten Wert. Laut Microsoft kann ein nicht authentifizierter Angreifer eine präparierte Kalendereinladung mit einem schädlichen Besprechungslink schicken; öffnet die Empfängerin oder der Empfänger die Besprechung und wählt den Link zum Teilnehmen, wird das Cross-Site-Scripting ausgelöst. Es braucht also eine Benutzerinteraktion, aber kein Konto in der Organisation.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) setzt nur ein Konto mit wenig Rechten und ein zugewiesenes Postfach voraus. Laut FAQ im Security Update Guide kann sich ein Angreifer durch Schwächen bei der Prüfung von Anfragen und Identity-Tokens als ein anderer Benutzer ausgeben und die Postfächer aller Exchange-Benutzer übernehmen: Mails lesen, senden und Anhänge herunterladen. Ein einziges kompromittiertes Benutzerkonto genügt als Ausgangspunkt.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) führt zum selben Ergebnis, der Übernahme aller Postfächer, setzt aber die Mitgliedschaft in einer hoch privilegierten Rollengruppe voraus.

Bei den beiden Remote-Code-Execution-Lücken ist die Ausgangslage unterschiedlich: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) verlangt ein authentifiziertes Konto mit geringen Rechten, [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) lässt sich ohne Anmeldung über einen präparierten Visio-Anhang auslösen, setzt laut Microsoft aber anhaltend knappen Arbeitsspeicher auf dem Zielsystem voraus. Die übrigen vier Lücken: CVE-2026-69378 (DoS durch unkontrollierte Rekursion, ohne Anmeldung), CVE-2026-69361 (SSRF, der Server sendet HTTP-Anfragen an interne oder Loopback-Systeme), CVE-2026-69375 (ein authentifizierter Angreifer kann Dateiinhalte ersetzen) und CVE-2026-69382 (Offenlegung von Anmeldedaten über einen schwachen Kryptoalgorithmus, setzt ein bereits erbeutetes Authentifizierungs-Cookie voraus).

## Version 2 vom 2. Oktober: CVE-2026-96940 nachgeliefert

Am 2. Oktober 2026 hat Microsoft «Version 2» der September-SUs veröffentlicht. Laut Ankündigung besteht der einzige Unterschied zur ersten Fassung im zusätzlichen Fix für **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)**. Diese Elevation-of-Privilege-Lücke hat CVSS 8.8, ist weder öffentlich bekannt noch ausgenutzt, wird von Microsoft aber als einzige der zehn CVEs mit **«Exploitation More Likely»** bewertet. Ein authentifizierter Angreifer kann damit auf fremde Postfächer derselben Organisation zugreifen und Mails samt Anhängen lesen. Exchange Online ist serverseitig bereits korrigiert.

| Version | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

Für die Praxis bedeutet das: Der Fix für CVE-2026-96940 steckt nur in den v2-Builds. Server, auf denen bereits das SU vom 8. September läuft, brauchen zusätzlich v2. Wer erst jetzt patcht, kann direkt v2 installieren, da SUs kumulativ sind. Ob Server mit dem ersten September-SU v2 zwingend installieren müssen, sagen die KB-Artikel nicht ausdrücklich; da die neue CVE nur mit v2 geschlossen wird, ist das die naheliegende Lesart.

## Wrapper-Problem behoben: SettingOverride jetzt entfernen

Das seit dem Juni-SU bekannte Problem, dass in Hybrid-Umgebungen *Wrapper-Nachrichten* im Posteingang freigegebener Postfächer auftauchen, ist mit dem September-SU auf allen vier Versionen behoben. Im [August-Artikel](/blog/exchange-security-updates-august-2026) stand noch, dass der als Workaround dokumentierte SettingOverride bestehen bleiben kann. Nach der Installation des September-SUs gilt das Gegenteil: Microsoft empfiehlt im zugehörigen Support-Artikel, den Override zu prüfen und zu entfernen.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Befehl | Wirkung |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Prüft, ob der Workaround-Override in der Organisation gesetzt ist. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Entfernt den Override, sobald das September-SU installiert ist. |

</details>

Meldet der erste Befehl, dass das Objekt `DisableBlockSharedAndUserMailboxHeaders` nicht gefunden wurde, ist laut Microsoft nichts weiter zu tun.

Für Exchange SE behebt das SU zusätzlich einen Fehler bei Hybrid-Frei/Gebucht-Abfragen über Microsoft Graph: On-Premises-Benutzer sahen die Belegtzeiten von Exchange-Online-Postfächern um ihren eigenen UTC-Versatz verschoben, ohne Fehlermeldung.

## Bekannte Probleme

**Veröffentlichte Kalender (.ics) liefern HTTP 500 an Kalender-Apps.** Das Problem besteht seit dem August-SU (Exchange SE ab Build 15.2.2562.46 sowie Exchange 2019 und 2016) und ist auch im September-SU und in v2 nicht behoben. Abonnements anonym veröffentlichter Kalender aktualisieren sich nicht mehr; im Browser funktioniert dieselbe URL. Ursache laut Microsoft: Exchange erkennt Clients am User-Agent, Kalender-Apps landen ohne Browser-Kennung in einem Codepfad, den das August-SU deaktiviert hat. Als Workaround beschreibt Microsoft eine URL-Rewrite-Regel auf der Site «Exchange Back End» im IIS, die bei .ics-Anfragen unter `/owa/calendar/` den Parameter `layout=premium` anhängt; dafür muss das IIS-Modul URL Rewrite installiert sein. Die genauen Schritte (über den IIS-Manager oder direkt in der `applicationHost.config`) stehen im [Support-Artikel KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Einen Termin für den Fix nennt Microsoft nicht (Stand 7. Oktober 2026).

**Frei/Gebucht für delegierte Postfächer in Hybrid-Umgebungen (nur Exchange SE).** Ist die Verfügbarkeitsabfrage ausschliesslich über die Graph API konfiguriert, schlagen Abfragen für Exchange-Online-Postfächer über delegierten On-Premises-Zugriff fehl. Outlook meldet «Your server location could not be determined», OWA zeigt «No information», in den EWS-Logs erscheint `(403) Forbidden`. Der dokumentierte Workaround leitet die Abfragen wieder über EWS statt über Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | Der Override, der die Weiterleitung der Verfügbarkeitsabfragen über Microsoft Graph steuert. |
| `-Parameters "Enabled=False"` | Schaltet den Graph-Pfad ab, die Abfragen laufen wieder über EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Richtet den Diagnoseaufruf an den Topologiedienst. |
| `-Component VariantConfiguration -Argument Refresh` | Lädt die Variant Configuration neu, damit der Override ohne Warten greift. |

</details>

In der Ankündigung zu v2 führt Microsoft dieses Problem unter den behobenen Problemen. Wer den Workaround gesetzt hat, sollte nach der Installation von v2 im [Support-Artikel KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) prüfen, ob er zurückgenommen werden soll; eine Anleitung dazu stand dort zum Stand 7. Oktober 2026 noch nicht.

**ContentEngine-Deadlock wegen fehlender koreanischer WordBreaker-Dateien (nur Exchange SE).** Mit dem September-SU (Build 15.2.2562.49 und v2-Build 15.2.2562.53) werden die Regeldateien für den aktualisierten koreanischen WordBreaker nicht mitinstalliert. Die Folgen: fehlende Suchergebnisse, verzögerte Mailzustellung, Outlook- bzw. MAPI-Clients, die hängen oder die Verbindung verlieren. Laut Ankündigung betrifft das Nachrichten in koreanischer Sprache. Der Workaround im [Support-Artikel KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) besteht darin, die beiden Dateien `ko.token.rule.bin` und `ko.complex.rule.bin` aus SQL Server 2025 Express RTM zu entnehmen, ihre SHA256-Hashes zu prüfen, sie in das Verzeichnis `Native` der Exchange-Installation zu kopieren und den Dienst Search Host Controller neu zu starten. Microsoft untersucht das Problem noch.

## Installation und Nachbereitung

Microsoft empfiehlt den bekannten Ablauf: mit dem [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) inventarisieren, bei veraltetem Stand mit dem [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) den Pfad ermitteln, das SU installieren, den Server neu starten und prüfen, ob alle Exchange-Dienste laufen. Der Health Checker zeigt danach auch, ob das SU korrekt installiert ist. Der Security Update Guide führt für die Updates einen erforderlichen Neustart.

Nach der Installation stehen drei Nacharbeiten an:

1. Den Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` entfernen, falls er gesetzt ist (siehe oben).

2. Auf Exchange SE prüfen, ob die Frei/Gebucht- und WordBreaker-Probleme auftreten, und gegebenenfalls die Workarounds anwenden.

3. Bei veröffentlichten Kalendern mit externen Abonnenten die URL-Rewrite-Regel aus KB5126672 einrichten, falls das noch nicht seit dem August-SU geschehen ist.

Aus dem Juli bleibt zudem die Kontrolle, ob die CVE-2026-42897-Mitigation (M2.1.0) noch aktiv ist; wie Sie sie entfernen, steht im [Artikel zum Juli-SU](/blog/exchange-security-updates-juli-2026).

## Empfohlenes Vorgehen

Installieren Sie auf allen Exchange-Servern und Management-Tools-Maschinen direkt die v2-Builds vom 2. Oktober; Server mit dem SU vom 8. September brauchen v2 für CVE-2026-96940 zusätzlich. Die Spoofing-Lücke mit CVSS 9.3 und die Postfachübernahme über ein Benutzerkonto mit geringen Rechten (CVE-2026-69380) sind Grund genug, nicht auf den nächsten Patchday zu warten. Entfernen Sie danach den Wrapper-Override, prüfen Sie die drei Known Issues und lassen Sie den Health Checker laufen. Für Exchange 2016 und 2019 endet das ESU-Programm im Oktober 2026; die Migration auf Exchange SE lässt sich nicht weiter verschieben.

## Quellen

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Offizielle Release-Ankündigung mit unterstützten Versionen, ESU-Hinweis, Known Issues, behobenen Problemen und Installationsablauf (eingesehen über den RSS-Feed des Exchange Team Blogs).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Ankündigung der v2 vom 2. Oktober 2026; einziger Unterschied ist CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): CVE-Liste, behobene Probleme und die drei Known Issues für Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): KB-Artikel für Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): KB-Artikel für Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): KB-Artikel für Exchange 2016 CU23, ohne CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Typ, CVSS, Schweregrad, Ausnutzungseinschätzung und FAQ zu allen neun September-CVEs und zu CVE-2026-96940, inklusive der betroffenen Builds je CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): KB-Artikel zur v2 für Exchange SE; die v2-Artikel für 2019 und 2016 sind KB5129956 bis KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Buildnummern der September-SUs und der v2 vom 2. Oktober 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Fix im September-SU und Anleitung zum Entfernen des SettingOverrides.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Ursache und URL-Rewrite-Workaround.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Symptome und SettingOverride-Workaround für Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Betroffene SE-Builds und manueller Workaround.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Der mit dem September-SU behobene Zeitzonenfehler in Exchange SE.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Deutschsprachige Aufschlüsselung der neun CVEs mit CVSS-Werten und Builds.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Deutschsprachige Zusammenfassung mit Hinweisen auf das spätere Ersatz-Update.
