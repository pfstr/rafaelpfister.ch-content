---
title: "Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-Ausfall 10/2026"
description: "Nach den Windows-Server-Updates vom 13. Oktober 2026 melden Exchange-Server nach rund 24 Stunden MapiExceptionMdbOffline (ec=1142): OWA schlägt fehl, Warteschlangen wachsen, DAG-Kopien replizieren nicht mehr. Ursache ist eine neue Grenze im RPC Endpoint Mapper; Microsoft empfiehlt seit dem 17. Oktober 2026 den Registry-Wert EpmapSavedContextCap, ein Fix fehlt noch. Symptome, betroffene KBs, Prüfbefehle und Vorgehen."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 Min. Lesezeit"
themen:
  - "exchange-updates"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange-updates"
  - "exchange-on-premises"
protokolle:
  - "troubleshooting"
  - "releases"
  - "powershell"
slug: "exchange-windows-updates-oktober-2026"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
url: "https://rafaelpfister.ch/blog/exchange-windows-updates-oktober-2026"
---

# Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober 2026: MapiExceptionMdbOffline (ec=1142)

Seit dem Patchday vom 13. Oktober 2026 fallen Exchange-Server nach rund 24 Stunden Laufzeit aus: Clients verlieren die Verbindung zum Postfach, OWA meldet `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, Mails bleiben in den Warteschlangen liegen, und in DAGs replizieren die Datenbankkopien nicht mehr. Microsoft hat die Ankündigung der Oktober-Exchange-SUs am 15. Oktober 2026 um einen Hinweis ergänzt: Auslöser sind die Oktober-Updates für Windows Server, die Installation des Exchange-SUs ist für das Problem nicht erforderlich. Seit dem 17. Oktober 2026 sind die Ursache und eine offizielle Gegenmassnahme per Registry bekannt; einen Fix gibt es noch nicht. Details im folgenden Abschnitt.

## Update 19. Oktober 2026: Ursache und Registry-Gegenmassnahme

Das Exchange-Team hat am 17. Oktober 2026 einen eigenen Beitrag zum Problem veröffentlicht. In den Release Notes der Windows-Updates (z. B. KB5126646) steht das Problem jetzt als Known Issue mit dem Status «ungelöst».

### Ursache

Die Oktober-Updates begrenzen den Speicher, den der RPC Endpoint Mapper für Endpoint-Lookup-Aufrufe verwendet. Standardmässig sind nur noch 512 gespeicherte Kontexte erlaubt; laut Microsoft reicht das für Lasten wie Exchange nicht. Exchange sammelt diese Kontexte im Betrieb an, bis die Grenze erreicht ist. Danach schlagen die RPC-Verbindungen zur Postfachdatenbank fehl, und OWA meldet `MapiExceptionMdbOffline`. Ein Leser bei Born hat den neuen Konfigurationswert `EpmapSavedContextCap` in der `RpcEpMap.dll` der Oktober-Updates nachgewiesen. Frankys Web ordnet die Grenze der am selben Patchday geschlossenen Denial-of-Service-Lücke CVE-2026-96300 im RPC Endpoint Mapper zu.

Damit sind mehrere Beobachtungen aus den ersten Tagen erklärt:

- Der Ausfall tritt erst nach Stunden auf, und der Zeitpunkt hängt von der Last ab. Laut Microsoft sind manche Server nie betroffen.
- Ein Neustart des Servers setzt den Zähler zurück und hilft deshalb nur, bis die Grenze erneut erreicht ist. Ein Neustart nur der Exchange-Dienste gibt den belegten Speicher laut einem Leser bei Born nicht frei.
- Die Grenze steckt in den kumulativen Windows-Updates. Exchange-SU und .NET-Update sind nicht beteiligt.
- Der vermutete Zusammenhang mit den Datenbank-Wartungsaufgaben und dem Replikationsdienst ist damit nicht die Ursache.

### Gegenmassnahme

Microsoft empfiehlt, die Grenze per Registry anzuheben und den Server anschliessend neu zu starten. In einer Eingabeaufforderung mit Administratorrechten:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Registry-Schlüssel der RPC-Konfiguration. |
| `/v EpmapSavedContextCap` | Name des Werts, der die Grenze für gespeicherte Endpoint-Mapper-Kontexte festlegt (ohne Eintrag gilt 512). |
| `/t REG_DWORD` | Datentyp: 32-Bit-Zahl. |
| `/d 65536` | Neue Grenze, wie von Microsoft empfohlen. |
| `/f` | Überschreibt einen vorhandenen Wert ohne Rückfrage. |

</details>

Der Endpoint Mapper liest den Wert nur beim Start; ohne Neustart des Servers wirkt die Änderung nicht. Tritt das Problem danach erneut auf, empfiehlt Microsoft, den Wert weiter zu erhöhen. Nach Microsofts Angaben hat die Massnahme das Problem in den bisher betreuten Kundenfällen gelöst. Laut Born bezeichnet Nino Bilic (Microsoft) den Eintrag als vorübergehende Abhilfe, die kein Sicherheitsrisiko darstellt, aber nicht die endgültige Lösung ist. Ein Leser bei Born meldet einen Server, der mit 65536 seit über 26 Stunden ohne Unterbrechung läuft. Einzelne Administratoren setzen höhere Werte wie 16777215; einen belegten Fall, in dem 65536 nach einem Neustart nicht ausgereicht hat, gibt es bisher nicht.

Ob der Wert gesetzt ist, zeigt diese Abfrage in PowerShell:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Registry-Schlüssel, in dem der Wert liegt. |
| `-Name EpmapSavedContextCap` | Liest nur diesen Wert; fehlt er, meldet PowerShell einen Fehler, und es gilt die Standardgrenze von 512. |

</details>

Setzen Sie den Wert auf allen Exchange-Servern, auch auf solchen, die bisher nicht ausgefallen sind. Auf Servern, die die Oktober-Updates noch nicht haben, können Sie den Wert vor der Installation setzen; der Neustart nach der Installation aktiviert ihn dann. In einer DAG starten Sie die Mitglieder wie unten beschrieben nacheinander neu. Frankys Web stellt zusätzlich ein PowerShell-Script bereit, das den Wert setzt, den bisherigen Zustand sichert und sich mit `-Remove` zurücknehmen lässt.

Die Windows-Updates sollen laut Microsoft ausdrücklich nicht deinstalliert werden. Ein Leser bei Frankys Web berichtet nach der Deinstallation von KB5126868 zudem von RDP-Anmeldeproblemen (CredSSP). Reicht die Registry-Massnahme nicht, bietet Microsoft über den Support for business eine weitere Gegenmassnahme an. Sobald ein Fix erscheint, sollten Sie den Registry-Wert wieder entfernen, weil die höhere Grenze den Schutz gegen CVE-2026-96300 abschwächt.

### Offene Punkte

- **Fix**: Microsoft nennt keinen Termin. Ein Leser bei Born vermutet, dass Microsoft die Grenze über ein Known Issue Rollback (Feature-ID 63096240) zurücknehmen könnte; bestätigt ist das nicht.
- **Wert**: Ob 65536 auch in grossen Umgebungen bis zum nächsten Patchday reicht, ist offen. Überwachen Sie die Event ID 1006 deshalb weiter.
- **Andere Serverrollen**: Ein Leser bei Born sieht auch Server ohne Exchange betroffen. Microsoft äussert sich nur zu Exchange.
- **Backup**: Ob der gemeldete VSS-Writer-Fehler eine Folge des Ausfalls ist, ist ungeklärt.

## Symptome

Die bisher berichteten Symptome:

- OWA schlägt mit `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)` fehl; im Application Log steht Event ID 1006.
- Outlook-Clients verlieren die Verbindung zum Postfach.
- Mails werden weder intern noch extern zugestellt, die Transport-Warteschlangen wachsen.
- In DAGs replizieren die Datenbankkopien nicht mehr; Switchover beim Wechsel in den Wartungsmodus schlagen laut Leserberichten bei Born fehl.
- Das Exchange Admin Center zeigt die Datenbanken trotzdem als gemountet an (Frankys Web).

Der Fehler tritt nicht direkt nach der Installation auf, sondern laut Berichten nach etwa 22 bis 30 Stunden Laufzeit. Ein Neustart des Servers stellt den Betrieb wieder her, der Ausfall wiederholt sich aber nach ungefähr derselben Zeit. In Kommentaren bei Born wird beobachtet, dass sich der Postfach-Replikationsdienst während oder nach den Datenbank-Wartungsaufgaben beendet und die MdbOffline-Fehler danach einsetzen; nach der inzwischen bekannten Ursache ist das eine Begleiterscheinung, nicht der Auslöser.

## Betroffene Updates

Microsoft nennt im Nachtrag zur Exchange-Ankündigung diese kumulativen Windows-Updates:

| Windows Server | Update |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

Am selben Patchday sind auch die Exchange-SUs (Exchange SE: KB5128895, «SU11») und kumulative Updates für das .NET Framework erschienen. Auf einem betroffenen Exchange SE unter Windows Server 2019 waren diese drei Pakete installiert:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft erwähnt das .NET-Update im Nachtrag nicht. Leser bei Born berichten aber von Ausfällen mit der Kombination aus Windows- und .NET-Update (dort KB5128928 unter Windows Server 2022) und von einem Server mit KB5126651 und Exchange-SU, jedoch ohne .NET-Update, der nach über 24 Stunden noch lief. Ob das .NET-Update beteiligt ist, ist damit offen.

Zum Exchange-SU: In Foren und Kommentaren heisst es, die Lage bessere sich, wenn das Exchange SU11 installiert ist. Das lässt sich nicht bestätigen; die Ausfälle traten auch auf einem Server mit SU11 auf. Umgekehrt berichten mehrere Administratoren von Ausfällen ohne installiertes Exchange-SU, was zu Microsofts Aussage passt. Das Exchange-SU schliesst sechs Schwachstellen, darunter eine kritische Remote-Code-Execution über eine präparierte E-Mail; Details im [Artikel zum Oktober-SU](/blog/exchange-security-updates-oktober-2026). Es zurückzuhalten, behebt das Problem nicht.

## Stand prüfen

Ermitteln Sie zuerst auf jedem Exchange-Server, welche der Oktober-Updates installiert sind:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `$kbs`, `-Id` | Liste der gesuchten KB-Nummern: die vier Windows-CUs aus dem Microsoft-Nachtrag und das .NET-Update für Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Unterdrückt die Fehlermeldung für KB-Nummern, die auf diesem Server nicht installiert sind. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Zeigt Server, KB-Nummer und Installationsdatum; aus dem Datum lässt sich abschätzen, wann der nächste Ausfall zu erwarten ist. |

</details>

`Get-HotFix` listet .NET-Updates nicht auf allen Systemen zuverlässig; im Zweifel zeigt der Updateverlauf in den Einstellungen den vollständigen Stand. Den Exchange-Build prüfen Sie in der Exchange Management Shell mit `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` und gleichen ihn mit den [Exchange-Buildnummern](/tools/exchange-builds) ab.

Anschliessend den Zustand von Datenbanken, Kopien und Warteschlangen in der Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Status aller Datenbankkopien auf dem lokalen Server; ohne DAG erscheint nur die aktive Kopie. |
| `CopyQueueLength`, `ReplayQueueLength` | Anzahl noch nicht kopierter bzw. noch nicht eingespielter Logdateien; steigende Werte zeigen, dass die Replikation steht. |
| `ContentIndexState` | Zustand des Suchindex der Kopie. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Zeigt nur Warteschlangen, die Mails enthalten. |
| `LastError` | Letzter Fehler der Warteschlange; bei diesem Problem typischerweise Verbindungsfehler zur Postfachdatenbank. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Sucht im Application Log nach Event ID 1006, wie von Microsoft genannt. |
| `StartTime=(Get-Date).AddDays(-2)` | Beschränkt die Suche auf die letzten zwei Tage, also auf die Zeit seit dem Patchday. |

</details>

Event ID 1006 protokollieren mehrere Exchange-Komponenten; entscheidend ist der Eintrag mit der MdbOffline-Meldung im Text.

## Vorgehen bis zum Fix

Die folgenden Punkte stammen vom 16. Oktober 2026, als es noch keine offizielle Gegenmassnahme gab. Seit dem 17. Oktober 2026 steht die Registry-Massnahme aus dem Update oben an erster Stelle; die Punkte 3 bis 6 bleiben als Rückfallebene gültig.

1. **Noch nicht aktualisierte Exchange-Server**: Ursprünglich lautete die Empfehlung, die Windows-Oktober-Updates zurückzustellen. Mit der Registry-Massnahme ist das nicht mehr nötig: Setzen Sie den Wert und installieren Sie die Updates, die ebenfalls Sicherheitslücken schliessen.

2. **Exchange-SU trotzdem installieren**: Das Exchange-SU KB5128895 (bzw. die ESU-Pakete für 2016/2019) ist laut Microsoft nicht der Auslöser und schliesst eine kritische Lücke, die ohne Konto per E-Mail ausnutzbar ist.

3. **Bereits betroffene Server**: Ein Neustart stellt den Betrieb für rund einen Tag wieder her. Planen Sie kontrollierte Neustarts in einem Abstand von deutlich unter 22 Stunden, zum Beispiel jede Nacht, statt auf den nächsten Ausfall zu warten. Ein Neustart nur der Exchange-Dienste hilft laut Lesern bei Born ebenfalls vorübergehend; wie lange, ist nicht belegt.

4. **DAG**: Starten Sie die Mitglieder nacheinander neu und verschieben Sie die aktiven Datenbanken vorher auf einen frisch gestarteten Server. Prüfen Sie nach jedem Neustart mit `Get-MailboxDatabaseCopyStatus`, ob die Kopien wieder `Healthy` sind und die Queues abgebaut werden, bevor Sie den nächsten Server neu starten.

5. **Warteschlangen überwachen**: Behalten Sie `Get-Queue` und die Event ID 1006 im Blick, idealerweise mit einer Alarmierung, damit ein Ausfall nicht erst durch Benutzermeldungen auffällt.

6. **Backup prüfen**: Ein Leser bei Born meldet nach dem Ausfall einen fehlgeschlagenen VSS-Writer im Backup. Kontrollieren Sie die Sicherungen der letzten Nächte.

Zur Deinstallation der Windows-Updates: Seit 2021 liefert Microsoft das kumulative Update zusammen mit dem Servicing Stack Update als kombiniertes Paket aus. `wusa /uninstall` entfernt ein solches Paket nicht; dafür ist `DISM /Online /Remove-Package` mit dem Paketnamen aus `DISM /Online /Get-Packages` nötig. Microsoft rät seit dem 17. Oktober 2026 ausdrücklich von der Deinstallation ab; mit der Registry-Massnahme besteht dafür auch kein Grund mehr.

Microsoft führt das Problem inzwischen als Known Issue in den Release Notes der Windows-Updates (Stand 19. Oktober 2026, ungelöst). Dieser Artikel wird nachgeführt, sobald ein Fix vorliegt.

## Quellen

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Nachtrag vom 15. Oktober 2026 mit den betroffenen Windows-Updates, Fehlermeldung, Event ID 1006 und der Aussage, dass das Exchange-SU für das Problem nicht erforderlich ist.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Ausfall nach etwa 24 Stunden, Datenbanken im EAC weiterhin als gemountet angezeigt, Neustart als vorübergehende Abhilfe, Kommentare zu DAG-Neustarts.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Leserberichte zu Zeitpunkt, beteiligten KBs (auch .NET), Replikationsdienst, Wartungsmodus und VSS-Fehlern.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Schweizer Zusammenfassung der Microsoft-Bestätigung.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): Beitrag des Exchange-Teams vom 17. Oktober 2026 mit dem Registry-Wert EpmapSavedContextCap, der Empfehlung, die Windows-Updates nicht zu deinstallieren, und dem Hinweis auf lastabhängige Auswirkungen.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue mit der Grenze von 512 Endpoint-Lookup-Aufrufen, Status ungelöst, weitere Gegenmassnahme über Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): Leseranalyse der RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), Aussage von Nino Bilic, Erfahrungen mit 65536 und höheren Werten.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): RPC Endpoint Mapper Service Denial of Service Vulnerability, CWE-770, veröffentlicht am 13. Oktober 2026.

9.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Inhalt des Exchange-SUs KB5128895 und der ESU-Pakete, Schwachstellen und Installationshinweise.
