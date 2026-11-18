---
title: "Nach dem Exchange-Online-Ausfall: SLA-Gutschrift bei Microsoft einfordern"
navTitle: "SLA-Gutschrift"
description: "Der Exchange-Online-Ausfall vom 12. und 13. November 2026 dauerte rund 22 Stunden und liegt damit für viele Tenants unter der zugesicherten Verfügbarkeit von 99,9 Prozent. Microsoft schreibt die Gutschrift nicht automatisch gut, sie muss bis zum 31. Dezember 2026 beantragt werden. Wie Sie die Verfügbarkeit berechnen, welche Belege Microsoft verlangt und warum kleine Firmen über ihren Partner und grosse Firmen direkt bei Microsoft vorgehen."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
featured: "2026-12-31"
slug: "microsoft-sla-gutschrift-einfordern"
translationId: "article-0e96d35615b0d8d9"
url: "https://rafaelpfister.ch/blog/microsoft-sla-gutschrift-einfordern"
---

# Nach dem Exchange-Online-Ausfall: SLA-Gutschrift bei Microsoft einfordern

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Kostenlose Unterstützung beim SLA-Antrag</p>
<p>Wir helfen Ihnen kostenlos, die Gutschrift für den Exchange-Online-Ausfall vom 12. und 13. November 2026 bei Microsoft oder Ihrem Partner zu beantragen: Verfügbarkeit berechnen, Belege zusammenstellen, Antrag einreichen. Die Frist läuft bis zum 31. Dezember 2026. Melden Sie sich über das <a href="https://rafaelpfister.ch/kontakt">Kontaktformular</a>.</p>
</div>

Vom 12. November 2026, 18:54 Uhr, bis zum 13. November 2026, 17:10 Uhr (Schweizer Zeit), hatten viele Organisationen keinen oder nur zeitweisen Zugriff auf ihre Exchange-Online-Postfächer. Microsoft führte die Störung im Admin Center unter EX1550543 (Exchange Online) und MO1550821 (weitere Microsoft-365-Dienste). Den Verlauf habe ich im [Liveticker zum Ausfall](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11) dokumentiert. Für Exchange Online sichert Microsoft eine monatliche Verfügbarkeit von 99,9 Prozent zu. Ein Ausfall von gut 22 Stunden liegt deutlich darunter. Eine Gutschrift erhalten Sie aber nur, wenn Sie sie beantragen, und der Weg dorthin hängt davon ab, wie Ihre Organisation Microsoft 365 eingekauft hat.

## Was das SLA zusichert

Massgebend ist das «Service Level Agreement for Microsoft Online Services», das Microsoft monatlich neu veröffentlicht. Dieser Artikel stützt sich auf die Fassung vom 17. November 2026. Für Exchange Online definiert das SLA eine Ausfallzeit als jede Zeitspanne, in der Benutzer mit Outlook im Web keine E-Mails senden oder empfangen können. Geplante Wartungsfenster gibt es für Exchange Online nicht.

Die Verfügbarkeit wird in Benutzerminuten gerechnet:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Die Benutzerminuten sind die Minuten des Monats mal die Zahl der Benutzer. Die Ausfallminuten sind die Dauer jedes Vorfalls mal die Zahl der davon betroffenen Benutzer. Unterschreitet die Verfügbarkeit eine Schwelle, gibt es eine Gutschrift auf die Monatsgebühr des betroffenen Dienstes:

| Verfügbarkeit im Monat | Gutschrift |
|---|---|
| unter 99,9 % | 25 % |
| unter 99 % | 50 % |
| unter 95 % | 100 % |

Für Microsoft Teams, SharePoint Online und OneDrive gelten dieselben Schwellen mit eigenen Definitionen der Ausfallzeit. Bei Teams zählt nur, wenn Benutzer keine Chats führen oder keine Besprechungen starten können. Während EX1550543 funktionierten Chats und Besprechungen laut Microsoft grösstenteils, gestört waren Kalender und Einladungen. Ein Teams-Antrag dürfte deshalb wenig Aussicht haben.

Exchange Online hat neben der Postfach-Verfügbarkeit weitere Service Levels, etwa die Zustellzeit innerhalb von Microsoft 365 und die Annahme eingehender Mails. Trifft ein Vorfall mehrere davon, müssen Sie sich laut SLA für einen entscheiden. Für den November-Ausfall ist die Postfach-Verfügbarkeit über Outlook im Web der naheliegende Massstab.

## Was der Ausfall in Prozent bedeutet

Der November 2026 hat 30 Tage, also 43'200 Minuten pro Benutzer. Daraus ergeben sich diese Grenzwerte pro Benutzer:

| Schwelle | Zulässige Ausfallzeit im November |
|---|---|
| 99,9 % | 43 Minuten |
| 99 % | 7 Stunden 12 Minuten |
| 95 % | 36 Stunden |

Die Störung dauerte von der ersten Meldung bis zur Behebung 1'336 Minuten. Zwei Beispielrechnungen für eine Organisation mit 200 Postfächern:

| Fall | Ausfallminuten | Verfügbarkeit | Gutschrift |
|---|---|---|---|
| Alle 200 Benutzer während der ganzen Störung ohne Zugriff | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 Benutzer während 10 Stunden ohne Zugriff | 120 x 600 = 72'000 | 99,17 % | 25 % |

Die Gesamtzahl der Benutzerminuten beträgt in beiden Fällen 200 x 43'200 = 8'640'000. Entscheidend ist, wie viele Benutzer tatsächlich wie lange betroffen waren. Microsoft prüft das anhand eigener Telemetrie, verlangt aber im Antrag Ihre Angaben dazu. Viele Tenants in Europa waren erst ab dem späten Abend des 12. November oder nur zeitweise betroffen. Rechnen Sie deshalb mit den Zeiten, die Sie belegen können, und nicht mit der Gesamtdauer der Microsoft-Meldung.

## Was die Gutschrift ist und was nicht

Das SLA regelt einige Punkte, die vor dem Antrag klar sein sollten:

1.  **Nur auf Antrag.** Microsoft prüft und gewährt Gutschriften nicht von sich aus. Eine Statusmeldung von Microsoft gilt laut SLA ausdrücklich nicht als Bestätigung, dass ein Service Level verfehlt wurde.

2.  **Gutschrift, keine Rückzahlung.** Die Gutschrift wird mit einer künftigen Rechnung verrechnet. Sie ist auf die Monatsgebühr des betroffenen Dienstes begrenzt.

3.  **Einziges Rechtsmittel.** Laut SLA sind Gutschriften das einzige Rechtsmittel für Verfügbarkeitsprobleme. Umsatzausfall, Überstunden oder andere Folgekosten ersetzt Microsoft nicht. Rechnungen dürfen Sie nicht eigenmächtig kürzen.

4.  **Nur der betroffene Dienst.** Haben Sie eine Suite wie Microsoft 365 E3 oder Business Premium, bezieht sich die Gutschrift auf den Anteil von Exchange Online, nicht auf den ganzen Lizenzpreis. Wie hoch dieser Anteil ist, legt Microsoft bei der Prüfung fest; das SLA nennt dafür keine Formel.

5.  **Keine Gutschrift für Gratis- und Testangebote.** Testversionen, Previews und kostenlose Dienste sind ausgeschlossen, ebenso gespendete Lizenzen für Non-Profit-Organisationen. Privatkunden mit Microsoft 365 Personal oder Family fallen nicht unter dieses SLA.

6.  **Open-Verträge in Tagen.** Wer über Open, Open Value oder Open Value Subscription lizenziert hat, erhält eine allfällige Gutschrift als zusätzliche Nutzungstage statt als Geldbetrag.

7.  **Frist.** Für Microsoft-365-Dienste muss der Antrag mit allen Angaben bis zum Ende des Monats eingehen, der auf den Vorfallmonat folgt. Für den November-Ausfall ist das der **31. Dezember 2026**. Für Azure gilt eine eigene Frist von 60 Tagen.

## Welche Angaben Microsoft verlangt

Das SLA nennt fünf Pflichtangaben. Fehlen sie, lehnt Microsoft den Antrag ab:

1.  eine genaue Beschreibung des Vorfalls,
2.  Zeitpunkt und Dauer der Ausfallzeit,
3.  die betroffenen Ressourcen (bei Exchange Online: Tenant und Dienst),
4.  Zahl und Standorte der betroffenen Benutzer,
5.  eine Beschreibung der aufgetretenen Fehler.

Die Kennung des Vorfalls (EX1550543) und die Tenant-ID gehören immer dazu. Beides lässt sich mit Microsoft Graph PowerShell abfragen:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | Berechtigung zum Lesen der Dienstintegritätsmeldungen |
| `-Scopes "Organization.Read.All"` | Berechtigung zum Lesen der Organisationsdaten, darunter die Tenant-ID |
| `(Get-MgOrganization).Id` | gibt die Tenant-ID aus |
| `-ServiceHealthIssueId "EX1550543"` | ruft genau diesen Vorfall ab |
| `Select-Object …` | zeigt Kennung, Titel, Beginn, Ende und Einstufung des Vorfalls |

</details>

Die Einstufung (`Classification`) zeigt, ob Microsoft den Vorfall als `incident` oder als `advisory` führt. Hinweise der Stufe «Advisory» sind nach Microsofts Partnerdokumentation in der Regel nicht gutschriftberechtigt. EX1550543 war als Incident eingestuft.

Für Zahl und Standort der Benutzer genügt eine Auswertung der lizenzierten Postfächer nach Nutzungsstandort:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `$abfrage = @{ … }` | sammelt die Parameter für `Get-MgUser` in einer Hashtable (Splatting), damit die Zeile kurz bleibt |
| `All` | liest alle Benutzer, nicht nur die erste Seite |
| `Filter 'assignedLicenses/$count ne 0'` | nur Benutzer mit mindestens einer zugewiesenen Lizenz |
| `ConsistencyLevel eventual` | für den `$count`-Filter erforderlich (erweiterte Abfrage) |
| `CountVariable anzahl` | legt die Gesamtzahl in der Variablen `$anzahl` ab |
| `Property UsageLocation` | lädt den Nutzungsstandort mit |
| `Group-Object UsageLocation` | zählt die Benutzer pro Land |

</details>

Diese Zahl ist die Obergrenze. Wie viele Benutzer tatsächlich keinen Zugriff hatten, belegen Sie mit eigenen Daten: Helpdesk-Tickets mit Zeitstempel, Fehlermeldungen aus Outlook und Outlook im Web (Screenshots), Ergebnisse von Verfügbarkeitstests aus Ihrem Monitoring sowie Anmelde- und Zugriffsprotokolle. Je genauer diese Angaben sind, desto weniger hängt das Ergebnis allein von Microsofts Telemetrie ab.

## Kleine Firmen: meist über den Partner

Kleine und mittlere Firmen kaufen Microsoft 365 in der Schweiz meist über einen Partner im Cloud Solution Provider Program (CSP) oder direkt mit Kreditkarte im Admin Center. Welcher Weg gilt, sehen Sie im Microsoft 365 Admin Center unter **Abrechnung > Ihre Produkte**: Steht dort ein Partner als Bezugsquelle, ist der Vertrag ein CSP-Vertrag.

### Gekauft über einen CSP-Partner

Microsoft nimmt den Antrag in diesem Fall nicht von Ihnen entgegen. Laut Partner Center dürfen nur CSP-Direktanbieter und indirekte Anbieter (Distributoren) Gutschriften beantragen, indirekte Reseller nicht. Ist Ihr IT-Dienstleister ein indirekter Reseller, läuft der Antrag also über ihn an seinen Distributor und von dort an Microsoft. Jede Stufe hat eigene, oft frühere Fristen. Melden Sie sich deshalb jetzt und nicht erst Ende Dezember.

Ihr Partner braucht von Ihnen:

1.  die Tenant-ID,
2.  die Kennung des Vorfalls (EX1550543),
3.  Zeitraum, Zahl und Standorte der betroffenen Benutzer sowie die beobachteten Fehler,
4.  eine E-Mail, in der Sie die SLA-Gutschrift beantragen. Sie muss von einer Adresse der betroffenen Domain kommen; eine Mail von einer privaten Adresse akzeptiert Microsoft nicht.

Den Vorfall selbst kennt Microsoft. Belegen muss der Antrag, dass Ihre Organisation betroffen war: Name, Tenant-ID und Ticketnummer allein reichen laut Microsoft nicht. Wird die Gutschrift gewährt, schreibt Microsoft sie dem Partner gut, und der Partner gibt sie an Sie weiter. Ob und wie er das tut, regelt Ihr Vertrag mit dem Partner. Fragen Sie nach, wenn auf der nächsten Rechnung nichts erscheint.

### Direkt bei Microsoft gekauft

Haben Sie Microsoft 365 direkt bei Microsoft bezahlt, stellen Sie den Antrag selbst im Admin Center. Dazu braucht es eine Administratorrolle:

1.  Unter `admin.cloud.microsoft` rechts unten **Hilfe und Support** öffnen.
2.  Das Anliegen beschreiben, zum Beispiel «Request an SLA service credit for a Microsoft 365 outage», und **Support kontaktieren** wählen.
3.  Als Kontaktweg E-Mail wählen. In den Titel gehören «SLA service credit request» und die Kennung EX1550543.
4.  In der Beschreibung die fünf Pflichtangaben aus dem vorherigen Abschnitt aufführen und die Belege anhängen.

Pro Dienst und Vorfall ist ein eigener Antrag nötig. Microsoft bearbeitet Anträge laut SLA in der Regel innert 45 Tagen.

### Lohnt sich das für kleine Firmen

Bei zehn Benutzern ist die Gutschrift klein: Bei einer Suite zählt nur der Exchange-Anteil, davon 25 oder 50 Prozent für einen Monat. Der Aufwand ist ebenfalls klein, wenn Sie die Angaben jetzt zusammenstellen. Der Antrag hat einen zweiten Nutzen: Er dokumentiert gegenüber Partner und Geschäftsleitung, wie stark Ihre Organisation betroffen war. Das hilft später bei der Frage, ob ein Fallback für E-Mail nötig ist.

## Grosse Firmen: direkt mit Microsoft, mit eigenen Messdaten

Grosse Organisationen lizenzieren Microsoft 365 meist über ein Enterprise Agreement (EA) oder das Microsoft Customer Agreement for Enterprise (MCA-E). Der Antrag geht hier direkt an Microsoft, entweder über das Admin Center wie oben beschrieben oder über den Unified-Support-Vertrag. Haben Sie einen Customer Success Account Manager (CSAM), informieren Sie ihn über den Antrag; er kann die Bearbeitung verfolgen. Bei einem EA über einen Licensing Solution Partner (LSP) klären Sie zusätzlich, auf welcher Rechnung die Gutschrift erscheint.

Bei mehreren tausend Benutzern geht es um nennenswerte Beträge, und Microsoft prüft die Angaben entsprechend genau. Der Unterschied zum kleinen Tenant liegt in der Beweisführung:

1.  **Betroffene Benutzer genau ermitteln.** Grosse Tenants verteilen sich auf mehrere Datenbanken und Regionen in Microsofts Rechenzentren, nicht alle Postfächer waren gleich lang gestört. Pauschal alle Benutzer als betroffen zu melden, ist bei grossen Tenants angreifbar. Belastbarer sind Messdaten: synthetische Tests gegen Outlook im Web pro Standort, Verfügbarkeitsdaten aus dem eigenen Monitoring, Auswertungen der Helpdesk-Tickets nach Standort und Zeit.

2.  **Standorte angeben.** Das SLA verlangt die Standorte der betroffenen Benutzer. Bei internationalen Organisationen lohnt sich die Aufteilung nach Land, weil die Störung zuerst Nord- und Südamerika und erst später andere Regionen traf.

3.  **Hybrid sauber abgrenzen.** Verzögerungen im eigenen Rechenzentrum, etwa Mails in der Warteschlange lokaler Exchange-Server zum Tenant, schliesst das SLA aus. Belegen Sie, dass der Fehler innerhalb von Microsoft 365 lag. Die Warteschlangen der lokalen Server sind dafür ein guter Beleg, weil sie zeigen, dass Exchange Online Mails nicht oder nur verzögert angenommen hat.

4.  **Den Service Level wählen.** Waren bei Ihnen vor allem die Zustellzeiten betroffen und weniger der Postfachzugriff, kann das Service Level für die Zustellzeit die bessere Grundlage sein. Pro Vorfall und Dienst ist nur ein Antrag zulässig, die Wahl sollte deshalb vor dem Antrag fallen.

5.  **Einkauf und Lizenzmanagement einbeziehen.** Die Gutschrift erscheint auf der Rechnung, nicht in der IT-Kostenstelle. Stimmen Sie mit dem Einkauf ab, wer den Antrag stellt, wer die Gutschrift kontrolliert und wie sie verbucht wird.

Grosse Organisationen sollten den Vorgang zudem für den nächsten Ausfall vorbereiten: eine Vorlage mit den fünf Pflichtangaben, ein Monitoring, das die Verfügbarkeit von Outlook im Web pro Standort misst, und eine klare Zuständigkeit für SLA-Anträge.

## Vorlage für den Antrag

Die folgende Vorlage enthält die Pflichtangaben aus dem SLA. Microsoft bearbeitet Anträge auf Englisch am schnellsten:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Quellen

1.  [Service Level Agreement for Microsoft Online Services (Übersicht der Fassungen)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): Fassung vom 17. November 2026, Definition der Ausfallzeit für Exchange Online, Gutschriftstufen, Pflichtangaben, Fristen und Ausschlüsse.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): Antragsweg für CSP-Partner, Ausschluss indirekter Reseller, erforderliche Belege, Einstufung von Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): Cmdlet zum Abrufen eines Dienstintegritäts-Vorfalls über Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): öffentliche Statusmeldungen zu EX1550543 und MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): Antragsweg im Admin Center für den November-Ausfall, Hinweis zu Privatkunden.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): Ausschluss von Privat-, Gratis- und Non-Profit-Lizenzen, ein Antrag pro Dienst und Vorfall.

7.  [Liveticker: Exchange Online, weltweiter Ausfall der Postfachzugriffe](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): Verlauf der Störung vom 12. und 13. November 2026 mit Microsoft-Chronik.
