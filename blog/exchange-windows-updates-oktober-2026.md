---
title: "Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Exchange-Ausfall 10/2026"
description: "Nach den Windows-Server-Updates vom 13. Oktober 2026 melden Exchange-Server nach rund 24 Stunden MapiExceptionMdbOffline (ec=1142): OWA schlägt fehl, Warteschlangen wachsen, DAG-Kopien replizieren nicht mehr. Microsoft bestätigt den Zusammenhang mit den Windows-Updates, ein Fix fehlt noch. Symptome, betroffene KBs, Prüfbefehle und Vorgehen."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 Min. Lesezeit"
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
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
---

# Exchange-Ausfälle nach den Windows-Server-Updates vom Oktober 2026: MapiExceptionMdbOffline (ec=1142)

Seit dem Patchday vom 13. Oktober 2026 fallen Exchange-Server nach rund 24 Stunden Laufzeit aus: Clients verlieren die Verbindung zum Postfach, OWA meldet `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, Mails bleiben in den Warteschlangen liegen, und in DAGs replizieren die Datenbankkopien nicht mehr. Microsoft hat die Ankündigung der Oktober-Exchange-SUs am 15. Oktober 2026 um einen Hinweis ergänzt: Auslöser sind die Oktober-Updates für Windows Server, die Installation des Exchange-SUs ist für das Problem nicht erforderlich. Einen Fix oder offiziellen Workaround gibt es noch nicht (Stand 16. Oktober 2026).

## Symptome

Die bisher berichteten Symptome:

- OWA schlägt mit `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)` fehl; im Application Log steht Event ID 1006.
- Outlook-Clients verlieren die Verbindung zum Postfach.
- Mails werden weder intern noch extern zugestellt, die Transport-Warteschlangen wachsen.
- In DAGs replizieren die Datenbankkopien nicht mehr; Switchover beim Wechsel in den Wartungsmodus schlagen laut Leserberichten bei Born fehl.
- Das Exchange Admin Center zeigt die Datenbanken trotzdem als gemountet an (Frankys Web).

Der Fehler tritt nicht direkt nach der Installation auf, sondern laut Berichten nach etwa 22 bis 30 Stunden Laufzeit. Ein Neustart des Servers stellt den Betrieb wieder her, der Ausfall wiederholt sich aber nach ungefähr derselben Zeit. In Kommentaren bei Born wird beobachtet, dass sich der Postfach-Replikationsdienst während oder nach den Datenbank-Wartungsaufgaben beendet und die MdbOffline-Fehler danach einsetzen; bestätigt ist das nicht.

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

Microsoft untersucht das Problem nach eigener Aussage zusammen mit dem Windows-Team und nennt bisher weder Workaround noch Rollback. Für die Zwischenzeit:

1. **Noch nicht aktualisierte Exchange-Server**: Stellen Sie die Windows-Oktober-Updates (und das .NET-Update vom selben Tag) auf Exchange-Servern zurück, in WSUS, Intune oder der eigenen Updateverwaltung. Das ist eine Abwägung: Die Windows-Updates schliessen ebenfalls Sicherheitslücken. Exchange-Server, die aus dem Internet erreichbar sind, sollten Sie deshalb besonders eng begleiten.

2. **Exchange-SU trotzdem installieren**: Das Exchange-SU KB5128895 (bzw. die ESU-Pakete für 2016/2019) ist laut Microsoft nicht der Auslöser und schliesst eine kritische Lücke, die ohne Konto per E-Mail ausnutzbar ist.

3. **Bereits betroffene Server**: Ein Neustart stellt den Betrieb für rund einen Tag wieder her. Planen Sie kontrollierte Neustarts in einem Abstand von deutlich unter 22 Stunden, zum Beispiel jede Nacht, statt auf den nächsten Ausfall zu warten. Ein Neustart nur der Exchange-Dienste hilft laut Lesern bei Born ebenfalls vorübergehend; wie lange, ist nicht belegt.

4. **DAG**: Starten Sie die Mitglieder nacheinander neu und verschieben Sie die aktiven Datenbanken vorher auf einen frisch gestarteten Server. Prüfen Sie nach jedem Neustart mit `Get-MailboxDatabaseCopyStatus`, ob die Kopien wieder `Healthy` sind und die Queues abgebaut werden, bevor Sie den nächsten Server neu starten.

5. **Warteschlangen überwachen**: Behalten Sie `Get-Queue` und die Event ID 1006 im Blick, idealerweise mit einer Alarmierung, damit ein Ausfall nicht erst durch Benutzermeldungen auffällt.

6. **Backup prüfen**: Ein Leser bei Born meldet nach dem Ausfall einen fehlgeschlagenen VSS-Writer im Backup. Kontrollieren Sie die Sicherungen der letzten Nächte.

Zur Deinstallation der Windows-Updates: Seit 2021 liefert Microsoft das kumulative Update zusammen mit dem Servicing Stack Update als kombiniertes Paket aus. `wusa /uninstall` entfernt ein solches Paket nicht; dafür ist `DISM /Online /Remove-Package` mit dem Paketnamen aus `DISM /Online /Get-Packages` nötig. Microsoft hat die Deinstallation als Workaround nicht freigegeben, und Berichte über einen erfolgreichen Rollback mit anschliessend stabilem Betrieb liegen bisher nicht vor. Testen Sie das, wenn überhaupt, zuerst auf einem einzelnen Server.

Laut Frankys Web führt Microsoft das Problem bei den Windows-Updates noch nicht als Known Issue (Stand 16. Oktober 2026). Behalten Sie die Microsoft-Ankündigung im Blick; Microsoft führt Ergänzungen dort am Ende unter «Updates to this blog post». Dieser Artikel wird nachgeführt, sobald ein Fix oder ein offizieller Workaround vorliegt.

## Quellen

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Nachtrag vom 15. Oktober 2026 mit den betroffenen Windows-Updates, Fehlermeldung, Event ID 1006 und der Aussage, dass das Exchange-SU für das Problem nicht erforderlich ist.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Ausfall nach etwa 24 Stunden, Datenbanken im EAC weiterhin als gemountet angezeigt, Neustart als vorübergehende Abhilfe, Kommentare zu DAG-Neustarts.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Leserberichte zu Zeitpunkt, beteiligten KBs (auch .NET), Replikationsdienst, Wartungsmodus und VSS-Fehlern.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Schweizer Zusammenfassung der Microsoft-Bestätigung.

5.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Inhalt des Exchange-SUs KB5128895 und der ESU-Pakete, Schwachstellen und Installationshinweise.
