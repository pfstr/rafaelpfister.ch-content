---
title: "Bis zu 10 Millionen Gratis-Tokens pro Tag: OpenAIs Data-Sharing-Programm mit Cost-Guardrails nutzen"
navTitle: "OpenAI-Gratis-Tokens"
description: "OpenAI schreibt Organisationen, die ihren API-Traffic zum Training freigeben, ein tägliches Gratis-Kontingent gut: je nach Tier bis zu 10 Millionen Tokens. Mit Prepaid-Guthaben, Projekt-Limits und einem Token-Budget im Code bleibt die Nutzung dauerhaft gratis."
date: "2026-08-27"
kategorie: "OpenAI API"
timeToRead: "9 Min. Lesezeit"
themen:
  - "openai-api"
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "openai-gratis-tokens-data-sharing"
translationId: "article-dde41cbe2dd858e6"
url: "https://rafaelpfister.ch/blog/openai-gratis-tokens-data-sharing"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
---

# Bis zu 10 Millionen Gratis-Tokens pro Tag: OpenAIs Data-Sharing-Programm mit Cost-Guardrails nutzen

OpenAI bezahlt für Trainingsdaten mit Rechenleistung statt mit Geld: Organisationen, die ihre API-Ein- und Ausgaben zum Training freigeben, erhalten seit Dezember 2024 ein tägliches Kontingent an Gratis-Tokens. Je nach Usage-Tier und Modellgruppe sind das zwischen 250'000 und 10 Millionen Tokens pro Tag. Für viele kleine Automatisierungen reicht das vollständig: Eine nächtliche Batch-Übersetzung, ein Klassifikations-Cronjob oder die automatische Verschlagwortung eines öffentlichen Archivs bleiben damit dauerhaft gratis.

Damit es gratis bleibt, braucht es Grenzen, und zwar an der richtigen Stelle. Ein Token-Zähler im eigenen Code ist eine Komfortfunktion; verbindlich sind nur die Limiten, die OpenAI selbst durchsetzt.

## Das Programm: Tokens gegen Trainingsdaten

Die Teilnahme läuft über die Einstellung **Share inputs and outputs with OpenAI** unter *Settings → Data controls → Sharing*. Ändern kann sie nur der Organization Owner, wahlweise für die ganze Organisation oder für einzelne Projekte. Wer für das Programm qualifiziert ist, sieht auf dieser Seite den Hinweis "You're eligible for free daily usage on traffic shared with OpenAI"; nach dem Aktivieren wechselt er zu "You're enrolled for complimentary daily tokens". Fehlt der Hinweis, ist die Organisation derzeit nicht teilnahmeberechtigt. Konten mit Zero Data Retention und Enterprise-Verträge sind vom Input-Output-Sharing ausgeschlossen.

Das Kontingent hängt vom Usage-Tier der Organisation ab und wird pro Modellgruppe gerechnet:

| Modellgruppe | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Grosse Modelle (u. a. gpt-5.6-sol, gpt-5.x, o-Serie, gpt-4.1, gpt-4o) | 250'000 Tokens/Tag | 1 Mio. Tokens/Tag |
| Kleine Modelle (u. a. gpt-5.6-terra, gpt-5.6-luna, mini- und nano-Varianten) | 2,5 Mio. Tokens/Tag | 10 Mio. Tokens/Tag |

Die wichtigsten Regeln im Detail:

- Gezählt werden Input- und Output-Tokens zusammen, geteilt über alle Modelle einer Gruppe. Der Zähler wird täglich um 00:00 UTC zurückgesetzt.
- Fine-getunte Modelle, Fine-Tuning-Training, Evals und Tool-Nutzung sind ausgeschlossen.
- Das Konto braucht ein positives Guthaben, sonst funktionieren auch die Gratis-Tokens nicht.
- OpenAI behält sich vor, das Programm zu beenden, mit 30 Tagen Vorlauf.

Die wichtigste Abrechnungsregel: Der Request, der das Tageskontingent überschreitet, wird **vollständig** zum Normaltarif berechnet, nicht nur der überschiessende Teil. Wer bei 975'000 von 1 Million Tokens steht und einen Request mit 30'000 Tokens schickt, bezahlt alle 30'000. Für die eigene Budgetplanung heisst das: Sicherheitsabstand einplanen, nicht auf das Kontingent hin optimieren.

## Was man dafür hergibt

Die Gegenleistung ist unmissverständlich: Alle Ein- und Ausgaben der freigegebenen Projekte gehen an OpenAI und können für das Training künftiger Modelle verwendet werden. Damit scheiden ganze Kategorien von Anwendungsfällen aus. Kundendaten, Support-Tickets, interne Dokumente, Code mit Konfigurationsdetails und alles mit Personenbezug dürfen nicht in ein freigegebenes Projekt gelangen; für Schweizer Firmen setzt hier bereits das revDSG die Grenze, bevor die Vertraulichkeit gegenüber Kunden überhaupt zur Sprache kommt.

Gut geeignet sind Workloads über Daten, die ohnehin öffentlich sind. Ein Beispiel ist die nächtliche Übersetzung eines öffentlichen Blogs in mehrere Sprachen: Die Artikel stehen im Netz, jeder Crawler liest sie heute schon, und die Übersetzungen werden ebenfalls veröffentlicht. Das Sharing gibt in einem solchen Fall nichts preis, was nicht schon preisgegeben wäre. Weitere Kandidaten sind Alt-Texte für ein öffentliches Bildarchiv, die Verschlagwortung von Open-Source-Dokumentation oder Zusammenfassungen öffentlicher Release Notes für ein Changelog.

## Cost-Guardrails im OpenAI-Konto einrichten

Die Reihenfolge ist bewusst gewählt: Zuerst kommen die Limiten, die OpenAI serverseitig durchsetzt. Sie greifen auch dann, wenn der eigene Code einen Fehler hat, ein Cronjob doppelt läuft oder ein Key in falsche Hände gerät.

**Prepaid-Guthaben, Auto-Reload aus.** Das Billing auf "Pay as you go" mit vorausbezahltem Guthaben stellen und die automatische Aufladung deaktivieren. Damit ist der maximale Schaden auf das Restguthaben begrenzt: Ist es aufgebraucht, lehnt die API weitere Requests ab. Da das Programm ein positives Guthaben voraussetzt, braucht es einen kleinen Sockel; 5 bis 10 Dollar genügen und bleiben bei sauberem Betrieb unangetastet. Dieser Schritt ist der einzige, der im schlimmsten Fall wirklich alles stoppt, deshalb steht er an erster Stelle.

**Ein dediziertes Projekt für den geteilten Traffic.** Das Sharing auf "Enabled for selected projects" stellen und nur ein eigens dafür angelegtes Projekt freigeben. Alle anderen Projekte der Organisation bleiben vom Training ausgenommen, und versehentlicher Traffic aus anderen Anwendungen landet weder im Trainingsdatensatz noch im falschen Budget.

**Projekt-Spend-Limit tief ansetzen.** Projekte haben ein eigenes monatliches Spend-Limit, und dieses ist hart: Requests schlagen fehl, sobald es erreicht ist. Für ein Projekt, das planmässig 0 Dollar kostet, darf es sehr tief liegen; 5 Dollar reichen als Reserve für den Fall, dass ein einzelner Lauf das Gratis-Kontingent überschreitet. Das Limit auf Organisationsebene ist dagegen als Obergrenze mit Alerts gedacht, die Warnschwellen (etwa bei 90 und 100 Prozent) lösen E-Mails aus.

**Ein restricted Key pro Projekt, nur als CI-Secret.** Der API-Key wird im Projekt erstellt, nicht auf Organisationsebene, und bekommt nur die Berechtigungen, die der Workload braucht. Für einen CI-Workflow heisst das: genau ein Key mit eingeschränkten Rechten, hinterlegt als Secret in der CI-Umgebung. Er taucht in keinem Repository, keiner lokalen Shell und keinem zweiten Dienst auf.

**Ein Modell aus der günstigen Gruppe wählen.** Der Unterschied zwischen den Gruppen ist der Faktor 10. Wer in Tier 1 arbeitet, hat mit einem Modell der kleinen Gruppe 2,5 Millionen Tokens pro Tag statt 250'000. Für strukturierte Aufgaben wie Übersetzung, Klassifikation oder Extraktion reicht die kleine Gruppe in aller Regel aus.

## Die zweite Verteidigungslinie im Code

Die Konto-Limiten verhindern finanziellen Schaden, aber sie führen zu harten Fehlern: Ein erreichtes Spend-Limit bricht den Lauf mitten im Batch ab. Wer sauber innerhalb des Gratis-Kontingents bleiben will, kann deshalb zusätzlich selbst mitzählen. Bewährt hat sich ein simpler Tageszähler, konfiguriert zum Beispiel so:

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

Der Mechanismus dahinter besteht aus vier Regeln:

- Nach jeder Antwort addiert der Job die von der API gemeldeten `input_tokens` und `output_tokens` auf einen Zähler in einer State-Datei. Es gibt keine Schätzung und keine zweite Abfrage, nur die Usage-Angaben aus der Antwort selbst.
- Vor jedem Request prüft er das Restbudget. Reicht es nicht mehr sicher für eine vollständige Antwort, endet der Lauf regulär mit dem Stop-Grund `token-budget` statt mit einem Fehler.
- Der Zähler arbeitet mit UTC-Kalendertagen und ist damit synchron zum Reset des Gratis-Kontingents um 00:00 UTC.
- Unabhängig vom Budget ist die Anzahl API-Calls pro Lauf gedeckelt, damit auch eine Serie fehlgeschlagener Versuche das Kontingent nicht aufbrauchen kann. Transport- und Quotenfehler brechen den Lauf ab, ohne automatische Wiederholung.

Das Budget dieses Beispiels liegt mit 1 Million bewusst deutlich unter dem Kontingent von 2,5 Millionen. Der Abstand folgt aus zwei Eigenheiten der Abrechnung. Erstens kennt der Zähler die Grösse des nächsten Requests nicht im Voraus; ein knapp kalkuliertes Budget kann deshalb um die Grösse eines Requests überschritten werden, und genau dieser Request würde nach der oben beschriebenen Regel vollständig berechnet. Zweitens liegen die Rate-Limits pro Tag (TPD) je nach Tier und Modell unterhalb des Gratis-Kontingents; ein Budget oberhalb des TPD-Limits würde nie regulär erreicht, weil vorher die API mit HTTP 429 ablehnt.

## Kontrolle: Das Dashboard muss 0.00 zeigen

Ob die Rechnung aufgeht, zeigt das Usage-Dashboard der Plattform. Zwei Ansichten genügen:

- Die **Usage**-Ansicht zählt alle Tokens, auch die gratis abgerechneten. Hier steht der volle Verbrauch des Workloads.
- Die **Costs**-Ansicht (und das Feld "Monthly spend" in der Projektliste) zeigt nur bezahlte Tokens. Hier muss dauerhaft 0.00 stehen.

Wer es genauer wissen will, kann die Usage-Ansicht nach *Service tier* gruppieren: Gratis verrechnete Tokens erscheinen dort als eigener Posten "data sharing incentive tier". Ein einmal pro Monat gesetzter Kalendereintrag für diesen Blick ins Dashboard schliesst die Kette der Guardrails ab, denn OpenAI kann das Programm mit 30 Tagen Frist beenden, und ab diesem Tag liefe derselbe Workload zum Normaltarif weiter.

## Quellen

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): massgebende Programmbeschreibung mit Modellgruppen, Tier-Kontingenten, UTC-Reset und der Abrechnungsregel für überschreitende Requests.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): Ankündigung der Programmverlängerung im April 2025 mit der Zusage der 30-Tage-Frist.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): Opt-in-Schalter und Enrollment-Status der eigenen Organisation (Login erforderlich).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): Erklärung der TPM-, RPM- und TPD-Limits, die neben dem Gratis-Kontingent gelten.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): Normaltarife, zu denen Überschreitungen des Kontingents abgerechnet werden.
