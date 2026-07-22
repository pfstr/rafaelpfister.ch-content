---
title: "Exchange Server Security Updates Juli 2026: CVE-2026-42897-Mitigation entfernen und Legacy-Sicherheitsgruppen aufräumen"
navTitle: "Exchange SU 07/2026"
description: "Microsoft hat am 14. Juli 2026 die Exchange Security Updates veröffentlicht. Zwei leicht übersehene Punkte: Die im Mai gesetzte CVE-2026-42897-Mitigation lässt sich jetzt sauber entfernen (samt der Stolperfalle, dass der Emergency-Mitigation-Service sie sonst wieder einspielt), und der Health Checker meldet neu zwei uralte, überprivilegierte Legacy-Sicherheitsgruppen im Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min to read"
themen:
  - "exchange-onprem-hybrid"
  - "active-directory-entra"
slug: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/blog/exchange-security-updates-juli-2026"
aiPrompt: |
  Du bist mein Exchange-Server-Administrationsassistent. Hilf mir, das Juli-2026-Security-Update sauber einzuspielen und nachzubereiten. Gehe Schritt für Schritt vor und berücksichtige zwei leicht übersehene Punkte:
  1. Die im Mai gesetzte CVE-2026-42897-Mitigation (IIS-URL-Rewrite-Regel M2.1.0) nach der Installation entfernen, ohne dass der Emergency-Mitigation-Service sie im nächsten stündlichen Lauf wieder einträgt.
  2. Mit dem Exchange Health Checker die deprecierten Sicherheitsgruppen "Exchange Domain Servers" und "Exchange Enterprise Servers" im Active Directory aufspüren und bereinigen.
  Frage nach den Werten, die nur ich kenne (Servernamen, CU-Stand, ESU-Programmstatus).
---

# Exchange Server Security Updates Juli 2026

Microsoft hat am 14. Juli 2026 die turnusmässigen Security Updates (SUs) für Exchange Server veröffentlicht. Neben den eigentlichen Sicherheitskorrekturen enthält das Release zwei Punkte, die in der Praxis leicht übersehen werden, aber wichtig sind: Man kann (und soll) jetzt die im Mai eingespielte Mitigation für **CVE-2026-42897** wieder entfernen, und der Exchange Health Checker prüft neu auf uralte, längst überflüssige Sicherheitsgruppen im Active Directory. In den offiziellen Blogposts werden beide Punkte nur kurz angerissen.

## Was im Juli-Release steckt

Die SUs stehen für folgende Versionen zur Verfügung:

- **Exchange Server Subscription Edition (SE) RTM**: als regulär verfügbares, öffentliches Update.
- **Exchange Server 2019 CU14 und CU15**: nur für Organisationen, die im **Period-2-ESU-Programm** eingeschrieben sind.
- **Exchange Server 2016 CU23**: ebenfalls nur über Period 2 ESU.

Exchange 2016 und 2019 sind out of support. Wer nicht im Period-2-ESU-Programm ist (gültig von Mai bis Oktober 2026), erhält diese Updates nicht mehr und sollte den Wechsel auf Exchange SE nicht länger aufschieben. Exchange-Online-Umgebungen sind bereits geschützt; in Hybrid-Setups muss das SU trotzdem auf allen Exchange-Servern eingespielt werden, auch auf reinen Management-Servern. Welche konkreten CVEs adressiert werden, steht wie üblich im Security Update Guide (Filter «Server Software» für Exchange SE bzw. «ESU» für 2016/2019).

Ein bekanntes Problem gibt es im aktuellen Release: In Hybrid-Umgebungen können sogenannte *Wrapper-Nachrichten* im Posteingang von freigegebenen Postfächern auftauchen. Details dazu im entsprechenden Microsoft-Support-Artikel.

## CVE-2026-42897-Mitigation nach der Installation entfernen

### Kurzer Rückblick

CVE-2026-42897 wurde am 14. Mai 2026 bekannt gegeben: eine Cross-Site-Scripting-Schwachstelle (Spoofing) in Outlook Web Access. Ein Angreifer schickt eine speziell präparierte E-Mail; öffnet das Opfer diese in OWA und sind bestimmte Interaktionsbedingungen erfüllt, lässt sich beliebiges JavaScript im Browser-Kontext ausführen. Betroffen waren Exchange 2016, 2019 und SE in *jedem* Patch-Stand. Microsoft veröffentlichte noch am selben Tag eine Notfall-Mitigation (ID **M2.1.x**, die konkrete IIS-Regel heisst **M2.1.0**) und lieferte mit dem Juni-2026-SU den eigentlichen Fix nach.

### Warum das Juli-Update die Mitigation *nicht* automatisch entfernt

Das ist der Punkt, der die meisten überrascht: Auch nach der Installation des Juli-SUs bleibt eine bereits angewandte Mitigation aktiv. Der Grund liegt in der Mechanik. Die Mitigation ist eine **Content-Security-Policy-basierte IIS-URL-Rewrite-Regel**, die *ausserhalb* des MSI-Installers eingespielt wurde, entweder durch die Emergency-Mitigation-Service (EM Service) oder durch das EOMT-Skript. Der MSI-Patch tauscht Binärdateien aus, verwaltet aber diese out-of-band gesetzten IIS-Regeln nicht. Deshalb ist das Entfernen ein eigener, manueller Schritt.

Nebenbei: Die Mitigation hat IE-Clients und Edge im IE-Modus ohnehin nie geschützt, weil der Internet Explorer keine CSP unterstützt. Wer solche Clients im Einsatz hatte, war über die Mitigation allein nie abgesichert. Das ist ein weiteres Argument, zeitnah zu patchen statt sich auf die Mitigation zu verlassen.

### Die Stolperfalle: der EM Service spielt die Mitigation wieder ein

Wer die Regel voreilig löscht, erlebt eine Überraschung. Der EM Service läuft stündlich und gleicht den Ist-Zustand mit den vom Office-Config-Service (Flighting) gelieferten Vorgaben ab. Die Zuordnung «welcher Build braucht welche Mitigation» liegt serverseitig. Erst eine serverseitige Änderung markiert den Juli-2026-Build als «Mitigation nicht mehr nötig». Diese Änderung wurde laut Microsoft erst rund um den 16. Juli 2026 vollständig ausgerollt. Bis dahin trägt der EM Service eine gelöschte M2.1.0-Regel im nächsten stündlichen Lauf einfach wieder ein.

Praktisch heisst das: Entweder wartet man mit dem manuellen Entfernen bis nach dem 16. Juli, oder man blockiert die Mitigation explizit, damit sie nicht reaktiviert wird.

### So entfernt man die Mitigation sauber (EM-Service-Pfad)

Zuerst prüfen, was überhaupt angewandt ist:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Um die Reaktivierung zu verhindern, wird die Mitigation-ID auf die Blockliste gesetzt: Einträge dort werden vom EM Service im stündlichen Lauf ignoriert.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

Danach die eigentliche IIS-Regel entfernen. Gut zu wissen und selten dokumentiert: Der EM Service legt seine URL-Rewrite-Regeln mit dem **Präfix «EEMS <Mitigation-ID> <Beschreibung>»** an. Damit findet man sie im IIS-Manager unter URL Rewrite (bzw. per `appcmd`/PowerShell in der `applicationHost.config`) eindeutig wieder, ohne raten zu müssen, welche Regel zur Mitigation gehört. Nach dem Ausrollen der serverseitigen Änderung kann man den Block wieder aufheben (`-MitigationsBlocked @()`), sofern man ihn nur als Übergangslösung gesetzt hat.

### EOMT-Pfad (getrennte oder Air-Gapped-Umgebungen)

Wurde die Mitigation über das herunterladbare **EOMT-Skript** (https://aka.ms/UnifiedEOMT) gesetzt, erfolgt der Rückbau über den Rollback-Schalter:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Auch hier ein wenig bekanntes Detail: EOMT sichert vor jeder Änderung den IIS-Ausgangszustand in einer **CVE-spezifischen JSON-Backup-Datei** unter `%WINDIR%\System32\inetsrv\config\`. Der Rollback liest genau diese Datei und stellt die ursprünglichen Einstellungen wieder her. Wichtig: Eine mit einem Legacy-Skript (EOMTv2 etc.) gesetzte Mitigation muss auch mit dessen eigenem Rollback-Mechanismus zurückgenommen werden: Die Backup-Formate sind nicht kompatibel.

### Warum sich das Entfernen lohnt

Die Mitigation ist nicht «gratis». Solange sie aktiv ist, schleppt man ihre bekannten Nebenwirkungen mit: Die OWA-Funktion «Kalender drucken» funktioniert nicht, Inline-Bilder werden im OWA-Lesebereich unter Umständen nicht korrekt angezeigt, OWA Light (`/?layout=light`) ist defekt (wird ohnehin demnächst abgeschaltet), veröffentlichte Kalender liefern teils Fehler 500. Besonders tückisch fürs Monitoring: Der Healthset **OWACalendar.Proxy** kann auf *unhealthy* springen und damit Fehlalarme in der Überwachung auslösen. Wer das SU installiert hat, aber die Mitigation stehen lässt, jagt am Ende Geistern hinterher. Sobald das Update installiert *und* die Mitigation entfernt ist, verschwinden auch diese Known Issues.

Ein Sonderfall: In gemischten Umgebungen dürfen noch nicht aktualisierte Server die Mitigation behalten. Man sollte aber wissen, dass die Office-Online-Server-Integration (OOS) unter Umständen erst dann wieder sauber funktioniert, wenn *alle* Exchange-Server in der Organisation auf dem Juli-Stand sind.

## Health Checker: uralte Sicherheitsgruppen aufspüren

Der zweite, vom SU-Release unabhängige Punkt: Der **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) prüft neu auf die Existenz zweier längst deprecierter Sicherheitsgruppen: **«Exchange Domain Servers»** und **«Exchange Enterprise Servers»**.

### Woher diese Gruppen kommen und warum sie ein Risiko sind

Diese beiden Gruppen stammen aus dem Berechtigungsmodell von Exchange 2000/2003 und sind seit Exchange 2007 deprecated. Mit Exchange 2007/2010 kam das Split-Permissions- bzw. RBAC-Modell, und seither werden sie schlicht nicht mehr verwendet. Das Problem: Verschwunden sind sie damit nicht. In vielen Verzeichnissen liegen sie seit rund zwei Jahrzehnten unbeachtet herum und tragen teilweise noch weitreichende ACLs aus dem alten Modell, also mehr Rechte, als eine moderne Exchange-Sicherheitsgruppe je hätte.

Genau das macht sie zum Angriffsvektor. Eine dormante Gruppe mit stehenden, breiten Berechtigungen ist eine klassische Eskalationskette: Wer es schafft, sich (oder ein kontrolliertes Konto) in eine solche Gruppe aufzunehmen, erbt deren Rechte im Verzeichnis. Da niemand die Gruppe aktiv beobachtet, fällt eine solche Manipulation kaum auf.

### Warum die meisten Admins sie nicht auf dem Schirm haben

Diese Gruppen sind aus mehreren Gründen ein blinder Fleck: Sie sind seit ~20 Jahren inaktiv, existierten meist schon vor der Amtszeit des heutigen Teams, überleben klaglos jede Migration und wurden vom Health Checker bisher nie angezeigt. Besonders heikel: Sie überstehen sogar die *vollständige* Ausserbetriebnahme von on-premises Exchange. Wer den letzten Exchange-Server entfernt hat, räumt in der Regel die Server-Objekte weg, übersieht aber diese Legacy-Gruppen komplett.

### Aufräumen

Der Health Checker meldet die Gruppen künftig automatisch. Manuell findet man sie im Active Directory (üblicherweise im `Users`-Container) bzw. per PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Vorgehen: Mitgliedschaft und allfällige benutzerdefinierte ACL-Referenzen prüfen, sicherstellen, dass nichts Produktives darauf verweist, und die Gruppen anschliessend löschen. Da sie seit 2007 deprecated sind, sind sie in der überwiegenden Mehrheit der Umgebungen gefahrlos entfernbar. Wer gar keinen on-premises Exchange mehr betreibt, sollte im gleichen Zug eine umfassendere AD-Bereinigung nach der offiziellen Microsoft-Anleitung einplanen.

Eine detaillierte Anleitung zum Entfernen der Gruppen hat Hayes Jupe in seinem Blogpost [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/) geschrieben.

## Empfohlenes Vorgehen

Kurz zusammengefasst der praktische Ablauf: Zuerst mit dem Health Checker den Bestand inventarisieren (er zeigt fehlende CUs/SUs, offene manuelle Schritte *und* neu die Legacy-Gruppen). Dann das aktuelle CU sowie das Juli-SU einspielen, den Server neu starten und prüfen, ob alle Exchange-Dienste sauber gestartet sind. Danach den Health Checker erneut laufen lassen, die CVE-2026-42897-Mitigation entfernen (nach dem 16. Juli bzw. mit vorherigem Block der ID M2.1.0) und schliesslich die deprecierten Sicherheitsgruppen bereinigen. SUs sind kumulativ: Wer auf einem unterstützten CU sitzt, muss nicht jedes Zwischen-SU nachziehen, sondern installiert direkt das neueste.

## Quellen

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146) — Offizielle Ankündigung des Juli-Releases mit den unterstützten Versionen und dem bekannten Wrapper-Nachrichten-Problem.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498) — Ursprüngliche Sicherheitsmeldung samt Notfall-Mitigation und den bekannten Nebenwirkungen in OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491) — Das Juni-Release, das den eigentlichen Fix für CVE-2026-42897 nachlieferte.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service) — Funktionsweise des EM Service, der Mitigationen stündlich abgleicht und eine voreilig gelöschte Regel wieder einträgt.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver) — Parameter `MitigationsApplied` und `MitigationsBlocked`, um Mitigationen zu prüfen und die Reaktivierung zu unterbinden.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/) — Das EOMT-Skript inklusive Rollback-Schalter und CVE-spezifischer JSON-Sicherung des IIS-Ausgangszustands.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897) — Technische Beschreibung und Bewertung der Schwachstelle in der National Vulnerability Database.
