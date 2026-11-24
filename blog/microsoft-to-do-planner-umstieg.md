---
title: "Microsoft To Do wird eingestellt: Umstieg auf Planner aus Admin-Sicht"
navTitle: "To Do wird eingestellt"
description: "Microsoft stellt die To-Do-Apps am 30. Juni 2027 ein. Geschäftskonten wechseln zu Planner, bestehende Aufgaben werden nicht migriert, und neue private Aufgaben speichert Planner ausserhalb des Exchange-Postfachs. Für Schweizer Tenants ändern sich damit Datenstandort und Compliance-Abdeckung; welche Benutzer betroffen sind, lässt sich per Microsoft Graph erheben."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-planner-umstieg"
translationId: "article-7691ba2aae63c942"
url: "https://rafaelpfister.ch/blog/microsoft-to-do-planner-umstieg"
---

# Microsoft To Do wird eingestellt: Umstieg auf Planner aus Admin-Sicht

Microsoft stellt Microsoft To Do ein. Die Apps für Web, Windows, Mac, iOS und Android werden am 30. Juni 2027 abgeschaltet, für Geschäfts- und Schulkonten ebenso wie für private Microsoft-Konten. Für Geschäfts- und Schulkonten übernimmt Microsoft Planner die persönlichen Aufgaben; die überarbeitete Ansicht «Meine Aufgaben» (My tasks) wird ab Dezember 2026 ausgerollt. Microsoft hat die Änderung im November 2026 in einem Support-Artikel mit Zeitplan, Unterschieden und Hinweisen für Administratoren veröffentlicht.

Kurzfazit: Eine automatische Migration gibt es nicht. Bestehende To-Do-Aufgaben bleiben bis zum 30. Juni 2027 in Planner sichtbar und liegen weiterhin im Exchange-Online-Postfach. Neue private Aufgaben speichert Planner ab dem Rollout in einem privaten Planner-Plan, also nicht mehr in Exchange. Damit ändern sich für Schweizer Tenants Datenstandort und Purview-Abdeckung. Prüfen Sie bis Mitte 2027 Lizenzen, geteilte Listen, Integrationen und Aufbewahrungspflichten. Private Microsoft-Konten erhalten keinen Ersatz.

## Zeitplan

| Zeitpunkt | Änderung |
|---|---|
| Dezember 2026 | Beginn des Rollouts der überarbeiteten Ansicht «Meine Aufgaben» in Planner für Geschäfts- und Schulkonten. Neu erstellte private Aufgaben speichert Planner ab dann in einem privaten Planner-Plan. |
| vor dem 30. Juni 2027 | Microsoft stellt eine Exportfunktion für To-Do-Daten bereit. Format, Umfang und Voraussetzungen sind noch nicht veröffentlicht. |
| 30. Juni 2027 | Die To-Do-Apps für Web, Windows, Mac, iOS und Android werden eingestellt. Bestehende To-Do-Aufgaben sind danach auch in Planner nicht mehr sichtbar. |
| später, noch ohne Datum | Der To-Do-Dienst, die APIs und die Synchronisation werden separat abgekündigt. Microsoft nennt die Termine gesondert. |

Planner ist für Geschäftskonten bereits heute im Web unter `planner.cloud.microsoft` und als App in Teams verfügbar. Microsoft kündigt an, die bestehenden Einstiegspunkte von To Do in Microsoft 365 anzupassen. Gegenüber Thurrott hat Microsoft präzisiert, dass die bestehenden Integrationen, darunter Outlook und Teams, auf Planner umgestellt werden.

## Bestehende Aufgaben werden nicht migriert

Microsoft überträgt bestehende To-Do-Aufgaben und -Listen nicht nach Planner. Bis zum 30. Juni 2027 lassen sie sich in Planner noch anzeigen und bearbeiten, danach nicht mehr. In den neuen privaten Plan werden sie nicht übernommen.

Gelöscht werden die Daten mit dem Ende der Apps nicht. Laut Microsoft plant die Abkündigung der Apps keine Löschung ein; Aufbewahrung und Zugriff behandelt Microsoft getrennt davon. To Do speichert Aufgaben als Exchange-Aufgaben im Exchange-Online-Postfach des Benutzers, im selben Postfach wie Mails, Termine, Kontakte und Notizen. Der Bestand bleibt deshalb nach dem 30. Juni 2027 vorerst im Postfach.

Das klassische Outlook ist von der Abkündigung nicht betroffen. Microsoft schreibt, dass dessen Aufgabenfunktion von To Do getrennt ist, weiter funktioniert und mit Exchange synchronisiert. To-Do-Listen erscheinen im klassischen Outlook als Aufgabenordner unter «Meine Aufgaben»; Benutzer mit klassischem Outlook können den Bestand dort also weiterhin lesen. Was mit den Daten geschieht, wenn Microsoft später den To-Do-Dienst selbst abschaltet, ist noch nicht beschrieben.

## Neue Aufgaben liegen ausserhalb von Exchange

Mit der neuen Ansicht ändert sich der Speicherort. Laut den FAQ für Planner-Administratoren liegen Aufgaben aus To Do und Outlook in Exchange, Pläne und ihre Aufgaben dagegen in Azure. Anhänge an Planaufgaben speichert Planner in der SharePoint-Site der Gruppe, Premium-Pläne in Dataverse. Neue private Aufgaben aus «Meine Aufgaben» liegen damit nicht mehr im Postfach des Benutzers. Für Administratoren hat das zwei Folgen.

### Datenstandort

Exchange Online gehört zu den Diensten mit dauerhafter Datenstandort-Zusage. Ist ein Tenant in der Schweiz angelegt, speichert Microsoft laut Product Terms Postfachinhalte (Mailtext, Kalendereinträge und Anhänge) ruhend nur in der Schweiz. Aufgaben nennt die Zusage nicht ausdrücklich, sie liegen jedoch im selben Postfach.

Planner führt Microsoft unter den weiteren Microsoft-365-Diensten ohne dauerhafte Zusage. Für Tenants mit Land Schweiz nennt die Tabelle als Speicherort «EUR1», die Makroregion Europa. Planner nimmt zudem nicht an Advanced Data Residency teil. Persönliche Aufgaben, die To Do bisher im Postfach in der Schweiz gespeichert hat, speichert Planner nach dem Wechsel in der Makroregion Europa. Die Speicherorte pro Dienst für Ihren Tenant sind im Microsoft 365 Admin Center auf der Datenstandort-Karte (Data Location Card) aufgeführt. Organisationen, die einen Datenstandort Schweiz vertraglich oder in internen Vorgaben zugesichert haben, müssen diese Änderung bewerten.

### Aufbewahrung und eDiscovery

Exchange-Aufbewahrungsrichtlinien in Microsoft Purview erfassen Aufgaben im Postfach nur, wenn sie ein Enddatum haben; Aufgaben ohne Enddatum sind nicht abgedeckt. Diese Einschränkung gilt bereits heute für To-Do-Aufgaben.

Für Planner unterstützt Purview eDiscovery-Suche und Legal Hold laut Microsoft nur für Aufgaben in Plänen, die mit einer Gruppe geteilt sind. Ausdrücklich nicht unterstützt sind persönliche Pläne (Roster-Pläne), Pläne in Loop-Komponenten, Premium-Pläne sowie Pläne in freigegebenen und privaten Teams-Kanälen. Aufbewahrungs- und Löschrichtlinien für Planner-Inhalte rollt Microsoft seit Mitte Juli 2026 aus, der Abschluss war für Anfang August angekündigt (MC1384412); laut Meldung lassen sie sich auf benutzer- und gruppenbasierte Pläne anwenden. Welchem Plantyp der neue private Plan technisch entspricht, geht aus der Ankündigung nicht hervor. Prüfen Sie nach dem Rollout, ob Ihre Aufbewahrungsrichtlinien und Holds die neuen privaten Aufgaben erfassen.

## Unterschiede zwischen To Do und Planner

Microsoft listet die folgenden Unterschiede auf:

| Bereich | Unterschied laut Microsoft |
|---|---|
| Organisation | Planner fasst private Aufgaben und zugewiesene Arbeit unter «Meine Aufgaben» zusammen, heute mit Raster- und Board-Ansicht. Eine Listenansicht folgt mit der Überarbeitung. |
| Erinnerungen | Die Benachrichtigungen zum Fälligkeitsdatum in Planner unterscheiden sich von den pro Aufgabe geplanten Erinnerungen in To Do. |
| Freigabe | Planner arbeitet mit Plänen und Zuweisungen. Mitgliedschaften, Zuweisungen und Berechtigungen geteilter To-Do-Listen werden nicht übernommen. |
| Desktop und offline | Planner gibt es im Web und in Teams. Native Apps für Windows und Mac sind nicht geplant; Web und Teams bieten keine vergleichbare Offline-Funktion. |
| Kategorien | Planner-Bezeichnungen funktionieren anders als Outlook-Kategorien und werden nicht mit ihnen synchronisiert. |
| Anhänge | Im neuen privaten Planner-Plan werden Dateianhänge an Aufgaben derzeit nicht unterstützt. |
| Stern und Priorität | Der persönliche Stern dient der eigenen Planung und kommt mit der neuen Ansicht. Die Priorität einer Aufgabe gilt für alle Beteiligten eines geteilten Plans. |
| Listengruppen | Ohne direkte Entsprechung in Planner. |
| Drucken, Liste per Mail senden | Für die Ersatzlösung derzeit nicht geplant. |

Benutzer, die mit der To-Do-App unter Windows oder Mac offline arbeiten, haben in Planner keinen gleichwertigen Ersatz.

## Lizenzen und Zugriff auf Planner

Planner ist unter anderem in Microsoft 365 Business Basic, Business Standard und Business Premium sowie in Office 365 E1, E3 und E5 enthalten. Fehlt Planner in einer Lizenz, bietet Microsoft Planner Plan 1 oder Planner and Project Plan 3 als eigene Abonnements an; diese sind auch für die Premium-Funktionen nötig. Eine neue Konfiguration ist laut Microsoft nicht erforderlich, und es gibt keinen eigenen Schalter, mit dem Administratoren den Wechsel steuern oder aufschieben können.

Ist Planner in der Lizenzzuweisung deaktiviert (Serviceplan `PROJECTWORKMANAGEMENT`), brauchen die betroffenen Benutzer wieder Zugriff, wenn Planner To Do ersetzen soll. Das Entfernen der Planner-Lizenz blendet laut Planner-FAQ ohnehin nur die Kachel aus: Über `planner.cloud.microsoft` können Benutzer ohne Lizenz weiterhin Pläne anlegen und bearbeiten. Haben Sie Planner bisher über die Lizenz gesperrt, prüfen Sie diese Entscheidung im Hinblick auf den Wechsel.

## Bestand erheben mit Microsoft Graph

Welche Benutzer To Do nutzen und welche Listen geteilt sind, lässt sich über die To-Do-API von Microsoft Graph auslesen. Das folgende Skript schreibt pro Benutzer alle Listen mit Freigabestatus und Anzahl offener Aufgaben in eine CSV-Datei. Es braucht die Module `Microsoft.Graph.Authentication` und `Microsoft.Graph.Users` sowie eine App-Registrierung mit den Anwendungsberechtigungen `Tasks.Read.All` und `User.Read.All` (mit Administratorzustimmung) und ein Zertifikat für die Anmeldung. `Tasks.Read.All` gewährt Lesezugriff auf die Aufgaben aller Benutzer; entfernen Sie die Berechtigung oder die App-Registrierung nach der Bestandsaufnahme wieder.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # Kein Exchange-Online-Postfach oder kein Zugriff: Benutzer überspringen
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `ClientId`, `TenantId` | App-Registrierung und Tenant, an dem sich das Skript anmeldet |
| `CertificateThumbprint` | Fingerabdruck des Zertifikats im Zertifikatsspeicher des aktuellen Benutzers; Anmeldung als App ohne Benutzerkonto |
| `-NoWelcome` | Unterdrückt die Begrüssungsmeldung von `Connect-MgGraph` |
| `-All` | Liest alle Seiten eines Ergebnisses statt nur der ersten |
| `-Filter "accountEnabled eq true"` | Nur aktivierte Konten; deaktivierte Konten wie freigegebene Postfächer entfallen |
| `-Property Id, UserPrincipalName` | Lädt nur die beiden benötigten Eigenschaften |
| `-UserId` | Benutzer, dessen Listen oder Aufgaben abgefragt werden |
| `-ErrorAction Stop` | Macht einen Fehler abfangbar, damit `catch` den Benutzer überspringen kann |
| `-TodoTaskListId` | Liste, deren Aufgaben abgefragt werden |
| `WellknownListName` | `defaultList` für die Standardliste «Aufgaben», `flaggedEmails` für gekennzeichnete Nachrichten, `none` für eigene Listen |
| `IsShared` | `True`, wenn die Liste mit anderen Benutzern geteilt ist |
| `IsOwner` | `False`, wenn die Liste einem anderen Benutzer gehört und nur geteilt wurde |
| `Status -ne 'completed'` | Zählt alle nicht erledigten Aufgaben |
| `-NoTypeInformation` | Keine Typzeile am Anfang der CSV-Datei |
| `-Encoding UTF8` | Umlaute in Listennamen bleiben erhalten |

</details>

Konten ohne Exchange-Online-Postfach liefern einen Fehler und werden übersprungen. In grossen Tenants ist ein Testlauf mit einer Teilmenge sinnvoll, etwa mit `-Top 20` statt `-All` bei `Get-MgUser`.

Die Auswertung der CSV-Datei:

- **Geteilte Listen** (`Geteilt` = `True`) zeigen, wo Zusammenarbeit über To Do läuft. Diese Mitgliedschaften überträgt Microsoft nicht; die Listen müssen als Planner-Plan mit den betreffenden Personen neu aufgebaut werden.
- **Offene Aufgaben** in eigenen Listen (`Typ` = `none` oder `defaultList`) müssen die Benutzer bis zum 30. Juni 2027 von Hand in Planner übernehmen oder mit der angekündigten Exportfunktion sichern.
- **Gekennzeichnete Nachrichten** (`Typ` = `flaggedEmails`) bleiben als Kennzeichnung an der Mail im Postfach erhalten.

## Private Microsoft-Konten

Auch für private Microsoft-Konten, einschliesslich Microsoft 365 Personal und Family, werden die To-Do-Apps am 30. Juni 2027 eingestellt. Planner steht für diese Konten nicht zur Verfügung. Microsoft empfiehlt, eine andere Aufgabenverwaltung zu wählen und die Daten mit der angekündigten Exportfunktion zu sichern. Ein Export bedeutet laut Microsoft nicht, dass sich die Daten in Planner oder eine andere Anwendung importieren lassen. Benutzer mit Geschäfts- und Privatkonto müssen beide Konten getrennt prüfen.

## Vorbereitung für Administratoren

1. **Zugriff und Lizenzen:** Stellen Sie sicher, dass alle To-Do-Benutzer eine Lizenz mit Planner haben und der Serviceplan `PROJECTWORKMANAGEMENT` aktiviert ist.

2. **Geteilte Listen und Integrationen:** Erheben Sie den Bestand mit dem Skript oben. Identifizieren Sie Power-Automate-Flows und Drittanwendungen, die die To-Do-API von Microsoft Graph verwenden. Microsoft kündigt die API separat und später ab; laut Microsoft können Integrationen trotzdem Anpassungen erfordern.

3. **Aufbewahrung und Datenstandort:** Prüfen Sie, ob bestehende Aufbewahrungspflichten für To-Do-Aufgaben gelten und ob Purview-Richtlinien und Holds den neuen privaten Plan erfassen. Bewerten Sie, ob der Speicherort in der Makroregion Europa mit Ihren Vorgaben vereinbar ist. Beziehen Sie vor dem Löschen oder Umorganisieren von Aufgabendaten die Compliance-Verantwortlichen ein.

4. **Benutzer informieren:** Teilen Sie den Benutzern mit, dass keine Migration stattfindet, wie lange der Bestand in Planner sichtbar bleibt und welche Funktionen sich ändern (Offline-Nutzung, Erinnerungen, Anhänge, Listengruppen).

5. **Message Center verfolgen:** Microsoft kündigt Rollout-Details, die Exportfunktion und die Termine für Dienst und APIs dort an.

## Quellen

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): Ankündigung mit Zeitplan, Unterschieden zwischen To Do und Planner, Vorbereitungsschritten und FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): Speicherung der To-Do-Aufgaben als Aufgaben im Exchange-Online-Postfach.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): Speicherorte von Aufgaben (Exchange, Azure, SharePoint, Dataverse), Serviceplan `PROJECTWORKMANAGEMENT` und Zugriff über `planner.cloud.microsoft` ohne Lizenz.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): Abonnements mit Planner sowie Planner Plan 1 und Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner ohne dauerhafte Datenstandort-Zusage, Speicherort «EUR1» für Tenants mit Land Schweiz.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): Zusage der Product Terms für ruhende Postfachinhalte in der Schweiz.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): Speicherung der Plandaten in Azure; Planner nimmt nicht an Advanced Data Residency teil. Stand August 2025, vor der Einführung der Aufbewahrungsrichtlinien für Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery und Legal Hold nur für Aufgaben in gruppenbasierten Plänen, Liste der nicht unterstützten Plantypen.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): Spiegel der Message-Center-Meldung zu Aufbewahrungs- und Löschrichtlinien für Planner, Rollout Juli bis August 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): unterstützte Postfachelemente; Aufgaben nur mit Enddatum.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): Ressourcen der To-Do-API in Microsoft Graph (Listen, Aufgaben, Checklisten, verknüpfte Ressourcen).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): Endpunkt, Berechtigungen und Eigenschaften `isShared`, `isOwner` und `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): Endpunkt für Aufgaben einer Liste und Anwendungsberechtigung `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): Bericht mit nachgereichter Stellungnahme von Microsoft zu den Integrationen in Outlook und Teams.
