---
title: "Exchange Online drosselt und blockiert ab September 2026 veraltete Exchange 2016 und 2019-Server: So funktioniert das Transport-Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Ab der zweiten Septemberwoche 2026 verlangt Exchange Online von Hybrid-Servern mindestens das Oktober-2025-SU, sonst wird der Mailfluss gedrosselt und später blockiert. Hintergründe zum Transport-Enforcement seit 2023, die Eskalationsstufen mit SMTP-Codes, der Report im Admin Center, die 90-Tage-Pause per PowerShell und warum die nächste Anhebung nur noch ESU-Kunden und Exchange SE durchlässt."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "exchange-updates"
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-transport-enforcement-hybrid-server"
translationId: "article-fff0c5efce59ef76"
url: "https://rafaelpfister.ch/blog/exchange-online-transport-enforcement-hybrid-server"
draft: false
---

# Exchange Online drosselt und blockiert ab September 2026 veraltete Exchange 2016 und 2019-Server: So funktioniert das Transport-Enforcement

Das Exchange-Team hat am 2. September 2026 angekündigt, die Mindestversion für Exchange 2016 und Exchange 2019 im Hybrid-Mailfluss anzuheben. Ab der zweiten Septemberwoche 2026 verlangt Exchange Online von Servern, die über einen Inbound Connector vom Typ `OnPremises` einliefern, mindestens den Stand des letzten öffentlichen Sicherheitsupdates vom Oktober 2025. Alles darunter wird gedrosselt und später blockiert. Kurzfazit: Wer seine Hybrid-Server seit Oktober 2025 nicht gepatcht hat, verliert in den nächsten Wochen schrittweise die Mailzustellung nach Exchange Online. Und die nächste Anhebung, die Microsoft für die kommenden Monate in Aussicht stellt, liegt oberhalb jedes öffentlich verfügbaren Updates: Dann erfüllen nur noch Kunden im kostenpflichtigen ESU-Programm oder Umgebungen mit Exchange Server Subscription Edition (SE) die Anforderung.

Die Ankündigung selbst ist kurz. Was sie in der Praxis bedeutet, ergibt sich aus dem Enforcement-System, das Microsoft seit 2023 schrittweise aufgebaut hat: Welche SMTP-Antworten Ihr Server zu sehen bekommt, wie Sie den Stand im Admin Center und per PowerShell prüfen und welche Optionen im Übergang bis zum Ende des ESU-Programms im Oktober 2026 bleiben.

## Was ab der zweiten Septemberwoche 2026 gilt

Die neue Untergrenze entspricht den Sicherheitsupdates vom 14. Oktober 2025. Das war der letzte Patchday, an dem Microsoft Updates für Exchange 2016 und 2019 öffentlich bereitgestellt hat; alle SUs seit Dezember 2025 gibt es nur über das ESU-Programm.

| Version | Mindeststand | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | Oktober-2025-SU (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | Oktober-2025-SU (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Für Exchange 2019 CU14 existiert ebenfalls ein Oktober-2025-SU (KB5066368, Build 15.2.1544.36). Im Microsoft-Beitrag ist als Mindestversion jedoch ausdrücklich CU15 SU5 genannt; CU14 ist ohnehin seit der Veröffentlichung von CU15 im Februar 2025 kein empfohlener Stand mehr. Planen Sie bei CU14 den Sprung auf CU15 mit ein.

Drei Eingrenzungen sind wichtig:

- **Nur der Hybrid-Mailfluss ist betroffen.** Exchange Online prüft die Version der einliefernden Server bei Nachrichten, die über einen Inbound Connector vom Typ `OnPremises` ankommen. Das ist die klassische Hybrid-Konfiguration, wie sie der Hybrid Configuration Wizard anlegt. Mails, die über ein Drittanbieter-Gateway oder einen Connector vom Typ `Partner` eintreffen, laufen nicht durch dieses Enforcement.
- **Die Version wird aus den Kopfzeilen gelesen.** Ein Exchange-Server schreibt seinen Build in die `Received`-Zeile jeder Nachricht, die er weitergibt (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online wertet diese Angabe aus. Damit zählt der Stand des Servers, der die Nachricht tatsächlich an Exchange Online übergibt, also in vielen Umgebungen der Edge Transport Server oder der Mailbox-Server mit dem Send Connector nach `*.mail.protection.outlook.com`.
- **Exchange SE ist nicht betroffen.** Das Enforcement gilt für Exchange 2016 und 2019; Exchange Server SE liegt oberhalb jeder Untergrenze, solange er regulär gepatcht wird.

## Hintergrund: das Transport-Enforcement seit 2023

Die Ankündigung vom September ist keine neue Massnahme, sondern die nächste Stufe eines Systems, das Microsoft im März 2023 unter dem Titel «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online» vorgestellt hat. Als «persistently vulnerable» definiert Microsoft jeden Exchange-Server, der entweder das Supportende erreicht hat oder für bekannte Schwachstellen ungepatcht bleibt. Ziel ist, Exchange-Online-Empfänger vor Nachrichten aus kompromittierbaren Servern zu schützen und gleichzeitig Druck auf die Betreiber auszuüben, die Server zu patchen oder abzuschalten.

Das System wurde versionsweise scharf geschaltet:

| Zeitpunkt | Betroffene Version |
|---|---|
| August 2023 | Exchange 2007 |
| September 2023 | Exchange 2010 |
| Dezember 2023 | Exchange 2013 |
| März 2024 | Exchange 2016 und 2019 (deutlich veraltete SU-Stände) |
| September 2026 | Exchange 2016 und 2019: Untergrenze = Oktober-2025-SU |
| «in einigen Monaten» | Exchange 2016 und 2019: Untergrenze oberhalb des letzten öffentlichen Updates |

Für Exchange 2016 und 2019 lag die Untergrenze bisher bei Ständen, die «significantly behind on security updates» waren. Neu ist, dass Microsoft die Grenze auf das letzte öffentliche Update setzt und damit erstmals Server trifft, die vor weniger als einem Jahr noch vollständig gepatcht waren.

## Die Eskalationsstufen

Das Enforcement arbeitet in drei Funktionen, die Microsoft «reporting», «throttling» und «blocking» nennt. Sobald ein Server unter die Untergrenze fällt, beginnt ein 90-Tage-Zyklus. Die Stufen aus dem Grundlagenbeitrag von 2023:

| Zeitraum | Massnahme | SMTP-Antwort |
|---|---|---|
| Tag 0 bis 30 | Nur Report im Exchange Admin Center | keine |
| Tag 30 bis 40 | Drosselung 5 Minuten pro Stunde | `450 4.7.230` |
| Tag 40 bis 50 | Drosselung 10 Minuten pro Stunde | `450 4.7.230` |
| Tag 50 bis 60 | Drosselung 20 Minuten pro Stunde | `450 4.7.230` |
| Tag 60 bis 70 | Drosselung 30 Minuten pro Stunde, zusätzlich Blockierung 5 Minuten pro Stunde | `450 4.7.230` und `550 5.7.230` |
| Tag 70 bis 80 | Blockierung 10 Minuten pro Stunde | `550 5.7.230` |
| Tag 80 bis 90 | Blockierung 20 Minuten pro Stunde | `550 5.7.230` |
| ab Tag 90 | Vollständige Blockierung | `550 5.7.230` |

Die beiden Antworten lauten im Wortlaut:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

Der Unterschied ist für den Betrieb entscheidend. Bei `450` lehnt Exchange Online die Verbindung temporär ab; der On-Premises-Server behält die Nachricht in seiner Warteschlange und versucht es erneut. Nutzer bemerken zunächst nur Verzögerungen, im Queue Viewer oder in `Get-Queue` wächst die Warteschlange zum Send Connector nach Exchange Online mit dem Status `Retry` und der 4.7.230-Meldung als `LastError`. Bei `550` ist die Ablehnung endgültig: Der Absender erhält einen NDR mit dem Code 5.7.230, die Nachricht ist verloren, sofern sie nicht erneut gesendet wird. Weil die Blockierung anfangs nur einige Minuten pro Stunde aktiv ist, wirkt das Fehlerbild zunächst sporadisch: Ein Teil der Nachrichten kommt an, ein Teil scheitert mit NDR. Wer ein solches Muster im Message Tracking sieht, sollte zuerst den Versionsstand prüfen, bevor er nach Netzwerk- oder Zertifikatsproblemen sucht.

Ob Microsoft für die neue Untergrenze den vollen 90-Tage-Zyklus ab der zweiten Septemberwoche startet oder bereits bei einer späteren Stufe einsteigt, sagt die Ankündigung nicht. Der Grundlagenbeitrag hält fest, dass das System nach einer Pause auf der zuvor erreichten Stufe weiterläuft. Verlassen Sie sich also nicht auf 30 Tage Schonfrist.

## Report im Exchange Admin Center und per PowerShell

Exchange Online listet die erkannten On-Premises-Server samt Version in einem eigenen Bericht: im Exchange Admin Center unter *Reports*, *Mail flow*, Bericht zu veralteten verbundenen On-Premises-Servern («out-of-date connecting on-premises Exchange servers»). Der Bericht zeigt pro Server den erkannten Build, ob er unter der Untergrenze liegt und in welcher Stufe sich das Enforcement befindet.

Dieselben Informationen liefert Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Befehl | Wirkung |
|---|---|
| `Connect-ExchangeOnline` | Öffnet die Sitzung zu Exchange Online (Modul `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Listet die von Exchange Online erkannten On-Premises-Server mit Build, Enforcement-Status und Stufe. |

</details>

Der Bericht kennt nur Server, die tatsächlich Mails an Exchange Online übergeben. Ein Management-Server ohne Mailfluss oder eine Maschine mit reinen Management Tools taucht nicht auf. Für das Enforcement ist das unerheblich, für die Sicherheit nicht: Auch diese Systeme brauchen die SUs.

## Enforcement pausieren: 90 Tage pro Jahr

Für Umgebungen, die die Untergrenze kurzfristig nicht erreichen, bietet Microsoft eine Pause an. Sie lässt sich für insgesamt 90 Tage pro Jahr aktivieren, am Stück oder in mehreren Abschnitten:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Get-TenantExemptionInfo` | Zeigt, ob und wie lange eine Pause für den Tenant aktiv ist. |
| `New-TenantExemptionInfo` | Legt eine neue Pause an. |
| `-BlockingScenario UnpatchedOnPremServer` | Wählt das Szenario «veralteter On-Premises-Server»; andere Szenarien gibt es für dieses Cmdlet derzeit nicht. |
| `-NumberOfDays 30` | Dauer der Pause in Tagen. Das Kontingent beträgt 90 Tage pro Jahr, die Angabe wird davon abgezogen. |

</details>

Zwei Eigenschaften der Pause sind in der Praxis wichtig. Erstens läuft das Enforcement nach Ablauf auf der Stufe weiter, auf der es angehalten wurde; die Pause setzt den 90-Tage-Zyklus nicht zurück. Zweitens gibt es kein Cmdlet, um eine laufende Pause vorzeitig zu beenden: Wer 90 Tage anlegt und nach zwei Wochen fertig patcht, hat das Jahreskontingent aufgebraucht. Legen Sie die Pause deshalb so kurz wie möglich an und verlängern Sie bei Bedarf.

Die Pause ist ausserdem nur für die aktuelle Untergrenze eine Lösung. Hebt Microsoft die Grenze in einigen Monaten über das letzte öffentliche Update, hilft ein aufgebrauchtes Kontingent nicht mehr.

## Warum die nächste Anhebung die eigentliche Frist ist

Exchange 2016 und 2019 sind seit dem 14. Oktober 2025 out of support. Microsoft hat danach zwei kostenpflichtige ESU-Perioden aufgelegt: Periode 1 bis April 2026, Periode 2 von Mai bis Oktober 2026. Mit der Ankündigung von Periode 2 am 15. April 2026 hat das Exchange-Team klargestellt, dass es keine weitere Verlängerung gibt. Die SUs von Dezember 2025 bis August 2026 (zuletzt Build 15.2.1748.49 für 2019 CU15 und 15.1.2507.72 für 2016 CU23) sind ausschliesslich für ESU-Kunden verfügbar und werden nicht öffentlich zum Download angeboten.

Damit ergibt sich folgende Lage:

- **Heute** erfüllt ein Server mit dem Oktober-2025-SU die Untergrenze, ob mit oder ohne ESU.
- **Bei der nächsten Anhebung** liegt die Untergrenze laut Microsoft oberhalb des Oktober-2025-Stands. Ohne ESU-Vertrag gibt es keinen legalen Weg, diesen Stand zu erreichen. Der Hybrid-Mailfluss dieser Server wird dann gedrosselt und blockiert, unabhängig davon, wie sauber der Rest der Umgebung betrieben wird.
- **Am 31. Oktober 2026** endet auch Periode 2. Danach gibt es für Exchange 2016 und 2019 keine SUs mehr, für niemanden. Spätestens die übernächste Anhebung trifft also auch ESU-Kunden.

Das ESU-Programm kauft damit im besten Fall wenige Monate. Der einzige dauerhafte Stand, den das Enforcement durchlässt, ist Exchange Server SE. Microsoft hat zudem angekündigt, dass Exchange SE CU2 (geplant für die zweite Jahreshälfte 2026) die Koexistenz mit Exchange 2016 und 2019 beendet: Die Installation bricht ab, wenn ältere Server in der Organisation gefunden werden. Die Migration ist also nicht nur wegen des Mailflusses fällig, sondern auch, um überhaupt weiter Updates für SE einspielen zu können.

Für Umgebungen, die Exchange On-Premises nur noch für die Verwaltung von Attributen in einer Hybrid-Konstellation behalten, ist die Alternative das Entfernen des letzten Servers: Seit Exchange 2019 CU12 lassen sich die Empfängerattribute mit den Management Tools ohne laufenden Exchange-Server pflegen. Dann gibt es keinen On-Premises-Mailfluss mehr und das Enforcement ist gegenstandslos.

## Versionsstand ermitteln

`Get-ExchangeServer` zeigt in `AdminDisplayVersion` nur das CU, nicht das SU. Verlässlich ist die Dateiversion von `ExSetup.exe` oder der [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), der zusätzlich fehlende manuelle Schritte meldet. Für einen schnellen Überblick über alle Server:

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Element | Wirkung |
|---|---|
| `Get-ExchangeServer` | Listet alle Exchange-Server der Organisation. |
| `\\<Server>\C$\...\ExSetup.exe` | Administrativer Freigabepfad zur Setup-Datei; bei abweichendem Installationspfad anpassen. |
| `VersionInfo.ProductVersion` | Dateiversion, die dem installierten SU-Build entspricht (z. B. `15.1.2507.61`). |

</details>

Liegt die Version unter `15.2.1748.39` (2019 CU15) beziehungsweise `15.1.2507.61` (2016 CU23), fällt der Server ab der zweiten Septemberwoche unter die Untergrenze.

## Empfohlenes Vorgehen

1. **Stand inventarisieren** wie oben beschrieben, inklusive Edge Transport Server und Management-Servern.

2. **Report in Exchange Online prüfen.** `Get-OnPremServerReportInfo` zeigt, welche Server Exchange Online tatsächlich sieht und ob bereits eine Enforcement-Stufe aktiv ist. Vergleichen Sie die Liste mit dem Inventar: Server, die dort fehlen, liefern nicht über den `OnPremises`-Connector ein.

3. **Mindestens das Oktober-2025-SU installieren.** KB5066367 (2019 CU15) und KB5066369 (2016 CU23) sind weiterhin öffentlich im Microsoft Download Center verfügbar. SUs sind kumulativ; ein Server auf dem August-2025-Stand kann direkt auf Oktober 2025 gehoben werden. Bei CU14 zuerst CU15 installieren. Nach der Installation Neustart, Dienststatus kontrollieren und den Health Checker erneut laufen lassen.

4. **Pause nur als Überbrückung nutzen.** Wenn das Update nicht in der ersten Septemberhälfte gelingt, `New-TenantExemptionInfo` mit knapper Laufzeit anlegen und die Pause nicht als Planungsreserve für die nächste Anhebung verstehen.

5. **Migration nach Exchange SE terminieren.** Ohne ESU-Vertrag ist die nächste Anhebung die harte Frist, mit ESU der 31. Oktober 2026. Exchange 2019 CU15 lässt sich per In-Place-Upgrade auf SE bringen; Exchange 2016 braucht den Umweg über eine SE-Neuinstallation und Postfach- beziehungsweise Rollenverschiebung. Wer Exchange nur noch für die Attributverwaltung betreibt, entfernt den letzten Server und arbeitet mit den Management Tools weiter.

## Quellen

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): Die Ankündigung vom 2. September 2026 mit der neuen Untergrenze (Oktober-2025-SU), dem Starttermin in der zweiten Septemberwoche und dem Hinweis auf die kommende Anhebung über das letzte öffentliche Update hinaus.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): Der Grundlagenbeitrag von 2023 mit der Definition «persistently vulnerable», den Stufen Reporting, Throttling und Blocking, dem 90-Tage-Zyklus, den SMTP-Antworten 4.7.230 und 5.7.230 sowie dem versionsweisen Rollout-Plan.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Die Cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` und `New-TenantExemptionInfo` samt Bericht im Exchange Admin Center und dem Jahreskontingent von 90 Tagen.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Build-Nummern der Oktober-2025-SUs und der nachfolgenden ESU-Updates bis August 2026; enthält auch den Hinweis, dass SUs ab Dezember 2025 nur ESU-Kunden erhalten.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): Der KB-Artikel zum Mindeststand für Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): Der KB-Artikel zum Mindeststand für Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): Das Supportende am 14. Oktober 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Bedingungen der ersten ESU-Periode.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Laufzeit Mai bis Oktober 2026 und die Aussage, dass keine weitere Verlängerung folgt.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Tabellarische Aufschlüsselung der acht Enforcement-Stufen und der Rollout-Daten je Exchange-Version; Drittquelle.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisierung von CU/SU-Ständen und offenen manuellen Schritten.
