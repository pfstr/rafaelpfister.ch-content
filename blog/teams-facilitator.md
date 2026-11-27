---
title: "Facilitator in Microsoft Teams: Was der Meeting-Agent kann und was nicht"
navTitle: "Teams Facilitator"
description: "Der Facilitator ist ein Copilot-Agent, der in geplanten Teams-Meetings gemeinsame Notizen schreibt, die Agenda führt, die Zeit überwacht und Fragen für alle sichtbar beantwortet. Der Artikel erklärt Funktionen, Lizenz- und Admin-Voraussetzungen, sinnvolle Einsatzszenarien und die Grenzen bei Meeting-Typen, Sprache und Compliance."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 Min. Lesezeit"
themen:
  - "microsoft-teams"
  - "microsoft-365-exchange"
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "teams-facilitator"
translationId: "article-33919f700e0a6587"
url: "https://rafaelpfister.ch/blog/teams-facilitator"
---

Seit Microsoft den Facilitator in Teams-Meetings ausrollt, taucht in vielen Tenants ein neuer Eintrag in den Meeting-Optionen auf: «Allow Copilot and Facilitator». Der Name lässt offen, was dahintersteckt. Der Facilitator ist ein KI-Agent, der als Teilnehmer im Meeting mitläuft und für alle sichtbar arbeitet. Daraus ergeben sich sowohl die Einsatzmöglichkeiten als auch die Grenzen.

## Was der Facilitator ist

Der Facilitator ist ein Agent von Microsoft 365 Copilot, der in einem geplanten Teams-Meeting die Rolle einer Moderationsassistenz übernimmt. Er schreibt während des Meetings Notizen in einer gemeinsamen Loop-Seite, die alle internen Teilnehmenden lesen und bearbeiten können. Er liest die Agenda aus der Einladung, zeigt einen Agenda-Timer auf der Meeting-Bühne und reagiert auf @Erwähnungen im Meeting-Chat.

Der Unterschied zu den anderen KI-Funktionen in Teams liegt in der Sichtbarkeit:

| Funktion | Wer sieht das Ergebnis | Zeitpunkt |
|---|---|---|
| Copilot in Teams-Meetings | nur die Person, die fragt | während und nach dem Meeting |
| Intelligent Recap | Teilnehmende mit Teams-Premium- oder Copilot-Lizenz | nach dem Meeting |
| Facilitator | alle internen Teilnehmenden | während des Meetings, Notizen auch danach |

Eine Frage an Copilot bleibt privat. Eine Frage an den Facilitator und seine Antwort erscheinen im Meeting-Chat für alle. Der Facilitator arbeitet damit für alle Teilnehmenden gemeinsam.

## Funktionen und Status

Microsoft führt die Funktionen in der Admin-Dokumentation getrennt nach allgemein verfügbar und Public Preview (Stand November 2026):

| Funktion | Status |
|---|---|
| Live-Notizen im Tab «Notes» (Loop-Seite, gemeinsam bearbeitbar) | allgemein verfügbar |
| Fragen beantworten aus dem Meeting-Inhalt und per Websuche | allgemein verfügbar |
| Agenda-Timer mit Hinweisen zur Halbzeit und vor Meeting-Ende | allgemein verfügbar |
| Word-Dokument aus dem Meeting erstellen | allgemein verfügbar |
| @Erwähnung des Facilitators im Meeting-Chat | allgemein verfügbar |
| Notizen auf dem Smartphone für Präsenzmeetings | allgemein verfügbar |
| Teams Rooms: Chat, Notizen und Timer im Raum | allgemein verfügbar |
| Aufgaben erfassen und mit Planner synchronisieren | Public Preview |
| PowerPoint-Präsentation aus dem Meeting erstellen | Public Preview |
| Notizen für ungeplante Präsenzmeetings in Teams Rooms | Public Preview |

Zur Agenda: Steht in der Einladung eine Traktandenliste, übernimmt der Facilitator sie, postet sie im Chat und hakt Punkte ab, sobald sie besprochen wurden. Das Abhaken erfolgt in Intervallen, nicht sofort. Läuft das Meeting über die geplante Endzeit, weist er darauf hin. Auf Wunsch setzt er auch einen Countdown für einen einzelnen Punkt.

Zu den Notizen: Der Facilitator fasst Entscheidungen, offene Fragen und Aufgaben zusammen. Grundlage ist ausschliesslich das gesprochene Wort über das Transkript. Was nur im Meeting-Chat steht, fliesst nicht in die Notizen ein. Die Notizen lassen sich jederzeit von Hand korrigieren und ergänzen; der Facilitator schreibt in dieselbe Seite weiter.

## Voraussetzungen

Für den Betrieb müssen mehrere Bedingungen gleichzeitig erfüllt sein:

- **Lizenz:** Wer den Facilitator zu einem Meeting hinzufügt oder während des Meetings einschaltet, braucht eine Microsoft-365-Copilot-Lizenz zusätzlich zu einer geeigneten Microsoft-365- und Teams-Lizenz. Teilnehmende ohne Copilot-Lizenz können dem Facilitator keine Fragen stellen, sehen aber die Notizen, die Fragen der anderen und seine Antworten.
- **Loop:** Die Loop-Erfahrungen in Teams müssen im Tenant eingeschaltet sein. Ohne Loop gibt es keine Notizen.
- **Transkription:** Beim Einschalten des Facilitators startet die Transkription automatisch. Steht die Meeting-Option «Allow Copilot and Facilitator» auf «Only during the meeting», läuft der Facilitator ohne gespeichertes Transkript; Fragen nach Meeting-Ende sind dann nicht möglich.
- **App-Freigabe:** Die App «Facilitator» ist standardmässig erlaubt, sofern im Teams Admin Center nicht alle Apps blockiert sind.

In der Einladung aktivieren Sie den Facilitator unter **Options** > **Copilot and other AI** mit der Einstellung «Allow Copilot and Facilitator». Alternativ lässt er sich während des Meetings über das Copilot-Menü einschalten.

## Wofür sich der Facilitator eignet

Den grössten Nutzen hat der Facilitator in Meetings, in denen sonst jemand Protokoll führt und dadurch nicht mitdiskutieren kann. Typische Beispiele:

- **Projekt- und Statusmeetings mit fester Traktandenliste:** Agenda-Timer und Hinweise zur Halbzeit halten den Zeitplan ein, die Notizen enthalten Entscheidungen und offene Punkte ohne separates Protokoll.
- **Workshops mit Entscheidungen:** Die gemeinsame Loop-Seite ist während des Meetings sichtbar. Die Teilnehmenden korrigieren Formulierungen direkt, statt ein Protokoll nachträglich per Mail abzustimmen.
- **Fragen während des Meetings:** «Was haben wir zu Punkt 2 entschieden?» oder eine kurze Websuche zu einer Produktversion beantwortet der Facilitator im Chat, sichtbar für alle.
- **Spätere Zusammenfassung:** Aus den Notizen erstellt der Facilitator auf Anfrage ein Word-Dokument, das in OneDrive der anfragenden Person abgelegt wird.
- **Präsenzmeetings:** Mit der Teams-App auf dem Smartphone oder in einem Teams Room werden auch Besprechungen vor Ort protokolliert.

## Was der Facilitator nicht kann

Die Grenzen sind in der Microsoft-Dokumentation klar beschrieben und betreffen vor allem Meeting-Typen, Sprache und Compliance.

**Nicht unterstützte Meeting-Typen.** Der Facilitator läuft nur in geplanten Teams-Meetings. Nicht unterstützt sind Sofortbesprechungen («Meet now»), aus einem Chat gestartete Meetings, Teams-Anrufe, Kanalbesprechungen sowie 1:1- und Gruppenchats. Für Webinare und Town Halls nennt die Dokumentation keine Unterstützung.

**Keine externen Teilnehmenden.** In Meetings und Chats mit externen Personen sind KI-generierte Notizen nicht verfügbar. Externe Teilnehmende sehen die Notizen grundsätzlich nicht. Für Meetings mit Kunden oder Lieferanten ist der Facilitator als Protokollwerkzeug damit nicht einsetzbar.

**Eine Sprache pro Meeting.** Beim Einschalten wird die gesprochene Sprache gewählt. Notizen entstehen nur für Passagen in dieser Sprache. In einem Meeting, das zwischen Deutsch und Französisch wechselt, fehlen die französischen Teile. Weicht die gewählte Sprache von der gesprochenen ab, entstehen keine Notizen. Schweizerdeutsch ist in der Liste der unterstützten Copilot-Sprachen nicht als eigene Sprache aufgeführt.

**Keine Aufzeichnung.** Der Facilitator schaltet die Transkription ein, aber keine Aufzeichnung. Wer Bild und Ton braucht, startet die Aufzeichnung separat.

**Keine Agenda aus geschützten Einladungen.** Ist die Einladung mit einer Vertraulichkeitsbezeichnung verschlüsselt, kann der Facilitator die Agenda nicht lesen. Dasselbe gilt, solange nur die organisierende Person im Meeting ist, weil er deren Kopie der Einladung nicht ausliest.

**Kein Ersatz für ein formelles Protokoll.** Die Notizen sind eine KI-Zusammenfassung des Transkripts. Für Verwaltungsratssitzungen, Personalgespräche oder andere Besprechungen mit rechtlicher Tragweite braucht es weiterhin ein geprüftes und freigegebenes Protokoll.

## Datenablage und Compliance

Die Notizen liegen als `.loop`-Datei im Ordner «Meetings» im OneDrive der Person, die den Facilitator gestartet hat. Microsoft behandelt sie wie Transkriptdaten. Für Compliance-Verantwortliche sind drei Einschränkungen relevant:

- **eDiscovery:** KI-generierte Meeting-Notizen werden in Microsoft Purview eDiscovery nicht automatisch als Cloud-Anlagen erfasst.
- **Aufbewahrung:** Aufbewahrungsbezeichnungen werden für Cloud-Anlagen in KI-generierten Notizen nicht unterstützt.
- **Vertraulichkeitsbezeichnungen:** Die Notizen übernehmen die Bezeichnung des Meetings nicht. Eine Bezeichnung lässt sich nachträglich auf die Loop-Komponente setzen. Einstellungen wie «Prevent copy and paste» und Wasserzeichen gelten nicht für Antworten und Notizen des Facilitators.

Für Organisationen mit Aufbewahrungspflichten bedeutet das: Notizen, die als Geschäftsunterlage gelten sollen, müssen in eine Ablage mit funktionierender Aufbewahrung überführt werden. Microsoft beschreibt die Purview-Aspekte in einer eigenen Seite (siehe Quellen).

## Facilitator steuern und deaktivieren

Im Teams Admin Center finden Sie die App unter **Teams apps** > **Manage apps** mit der Suche nach «Facilitator». Dort lässt sie sich für den ganzen Tenant erlauben oder blockieren. Für einzelne Benutzergruppen steuern Sie die App über App-Berechtigungsrichtlinien oder App-zentrierte Verwaltung.

Die Loop-Notizen in Meetings, auf denen der Facilitator aufbaut, hängen an einer SharePoint-Tenant-Einstellung. Diese gilt für alle Benutzer und lässt sich nicht pro Person setzen:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Url` | Adresse des SharePoint-Admin-Centers des Tenants (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` schaltet die gemeinsamen Meeting-Notizen und damit die KI-Notizen des Facilitators für alle Benutzer aus, `$true` wieder ein |

</details>

Diese Einstellung entfernt die Notizfunktion auch für Meetings ohne Facilitator. Soll nur der Agent gesperrt werden, ist das Blockieren der App im Teams Admin Center der gezieltere Weg. Für Meetings ohne gespeichertes Transkript setzen Sie in der Meeting-Richtlinie die Option für Copilot und Facilitator auf «Only during the meeting».

## Quellen

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Admin-Dokumentation mit Lizenzanforderungen, Funktionsstatus, Datenablage, Compliance-Einschränkungen und App-Steuerung (Stand 18.11.2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Endbenutzer-Anleitung zu Meeting-Optionen, Agenda, Timer, unterstützten Meeting-Typen und Sprachwahl.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Präsenzmeetings in Teams Rooms on Windows, Lizenz der organisierenden Person, Zuordnung von Sprechenden im Raum.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Microsoft-365-Insider-Blog vom Oktober 2025 zu Transkription, Notizgrundlage und Betrieb ohne gespeichertes Transkript.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Purview-Sicht auf Facilitator-Daten für Aufbewahrung und eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Tenant-Einstellungen `IsLoopEnabled` und `IsCollabMeetingNotesFluidEnabled` für Loop-Komponenten und Meeting-Notizen.
