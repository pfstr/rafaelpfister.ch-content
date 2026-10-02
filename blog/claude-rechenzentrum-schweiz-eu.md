---
title: "In welchem Rechenzentrum rechnet Claude? Verbindung nachverfolgen und Verarbeitung auf Schweiz oder EU beschränken"
navTitle: "Claude-Rechenzentrum"
description: "Welche Stationen eine Anfrage an Claude durchläuft, warum die Spur am Cloudflare-Edge endet und was Anthropic über seine Rechenzentren offenlegt. Dazu der Vergleich der Wege mit festem Verarbeitungsort (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), die Grenzen für einen reinen Schweiz-Betrieb und was die regionale Bindung kostet."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min to read"
themen:
  - "claude"
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "claude-rechenzentrum-schweiz-eu"
url: "https://rafaelpfister.ch/blog/claude-rechenzentrum-schweiz-eu"
translationId: "article-d8b7299589ea96f4"
aiPrompt: |
  Du bist mein Berater für Datenstandorte bei KI-Diensten. Hilf mir Schritt für Schritt zu entscheiden, über welchen Weg (Anthropic API, AWS Bedrock, Google Vertex AI oder Microsoft Foundry) wir Claude nutzen sollen, wenn die Verarbeitung in der Schweiz oder in der EU bleiben muss. Frage mich zuerst nach Anwendungsfall, Datenklassifizierung, benötigtem Modell, erwartetem Token-Volumen pro Monat und bestehendem Cloud-Anbieter. Berechne danach die Monatskosten für den globalen und den regional gebundenen Endpunkt und nenne die konkreten Konfigurationsschritte (Region, Inference Profile, Endpoint, Kontrollmöglichkeiten im Log).
---

Wer Claude über claude.ai, Claude Code oder die API nutzt, erfährt nicht, in welchem Rechenzentrum das Modell die Anfrage verarbeitet. Sichtbar ist nur der erste Abschnitt der Verbindung bis zum nächsten Edge-Knoten. Für Unternehmen, die Personendaten oder vertrauliche Inhalte an ein Sprachmodell übergeben, ist das zu wenig: Sie müssen gegenüber Kunden, Datenschutzberatern oder der Revision belegen können, wo die Verarbeitung stattfindet.

Dieser Beitrag zeigt, wie weit sich die Verbindung selbst nachverfolgen lässt, was über Anthropics Rechenzentren öffentlich bekannt ist und über welche Wege sich die Verarbeitung verbindlich auf die EU oder den Raum EU plus Schweiz beschränken lässt. Die Preise und Regionen entsprechen dem Stand vom 2. Oktober 2026.

**Schweiz-Fokus:** Massgebend ist das revidierte Datenschutzgesetz (revDSG). Eine Bekanntgabe von Personendaten ins Ausland ist nach Art. 16 revDSG zulässig, wenn der Bundesrat dem Zielstaat einen angemessenen Schutz attestiert (Anhang 1 der Datenschutzverordnung, DSV) oder geeignete Garantien wie Standardvertragsklauseln bestehen. Die EU- und EWR-Staaten stehen auf dieser Liste, die USA nur für Unternehmen, die nach dem Swiss-U.S. Data Privacy Framework zertifiziert sind. Ein Verarbeitungsort in der EU vereinfacht die Begründung deshalb erheblich, ersetzt aber nicht die übrigen Pflichten (Auftragsbearbeitungsvertrag, Information der betroffenen Personen, Datensicherheit).

**EU-Hinweis:** Für Unternehmen mit Sitz in der EU gelten stattdessen die Art. 44 ff. DSGVO. Die EU-Kommission hat der Schweiz ein angemessenes Datenschutzniveau bescheinigt (Angemessenheitsbeschluss 2000/518/EG, im Januar 2024 bestätigt); eine Verarbeitung in Zürich ist für EU-Unternehmen also kein Drittlandtransfer im problematischen Sinn.

## Welche Stationen eine Anfrage an Claude durchläuft

Eine Anfrage an `claude.ai` oder `api.anthropic.com` läuft über drei Stationen:

| Station | Wer betreibt sie | Für Sie sichtbar? |
|---|---|---|
| Edge-Knoten (TLS-Terminierung, Schutz vor Missbrauch) | Cloudflare, im Namen von Anthropic | Ja, Standort per Header ablesbar |
| Internes Netz bis zum Backend | Cloudflare und Anthropic | Nein |
| Inferenz (das Modell rechnet) | Anthropic, auf Rechenleistung von AWS, Google Cloud und eigenen Rechenzentren | Nein, nur grobe Angabe `us` oder `global` |

Die Hostnamen `claude.ai` und `api.anthropic.com` zeigen beide auf die Adresse `160.79.104.10`. Sie gehört zum Präfix `160.79.104.0/23`, das auf Anthropic registriert ist (AS399358) und laut RPKI gültig von Anthropic stammt. Im globalen Routing wird das Präfix jedoch ausschliesslich über Cloudflare (AS13335) angekündigt: Anthropic bringt seine eigenen IP-Adressen in das Anycast-Netz von Cloudflare ein. Dieselbe Adresse wird dadurch weltweit von jedem Cloudflare-Standort beantwortet, und die Verbindung landet beim nächstgelegenen Edge-Knoten.

## Den Edge-Knoten selbst bestimmen

Cloudflare gibt den bedienenden Standort in einer Diagnoseseite und im Header `cf-ray` preis. Das Kürzel hinter der Ray-ID ist ein IATA-Flughafencode: `ZRH` steht für Zürich, `GVA` für Genf, `FRA` für Frankfurt, `MRS` für Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

Relevant sind in der Ausgabe die Zeilen `colo=` (Edge-Standort) und `loc=` (Land, dem Cloudflare Ihre Absenderadresse zuordnet). Unter Windows liefert PowerShell dasselbe:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Den Header lesen Sie mit einer HEAD-Anfrage aus:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-s` | Unterdrückt Fortschrittsanzeige und Fehlermeldungen von curl |
| `-I` | Sendet eine HEAD-Anfrage und gibt nur die Antwort-Header aus |
| `grep -i` | Sucht ohne Beachtung der Gross-/Kleinschreibung |
| `-E '^(server\|cf-ray):'` | Erweiterter regulärer Ausdruck: nur Zeilen, die mit `server:` oder `cf-ray:` beginnen |

</details>

Aus einem Schweizer Netz ist in der Regel `ZRH` oder `GVA` zu erwarten. Liegt der Internetausgang an einem anderen Ort, etwa bei einem Firmennetz mit zentralem Ausgang im Ausland, antwortet der Edge-Knoten dort, zum Beispiel mit `colo=FRA` oder `colo=MRS`. Der Edge-Standort folgt also Ihrem Netzausgang, nicht dem Ort der Verarbeitung.

## Warum der Traceroute am Edge endet

Ein Traceroute zeigt den Weg bis zum Edge-Knoten und nicht weiter:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-ComputerName api.anthropic.com` | Zielhost der Messung |
| `-TraceRoute` | Ermittelt die Router-Hops auf dem Weg zum Ziel |
| `-Hops 20` | Maximale Anzahl Hops, die geprüft werden |

</details>

Unter Linux und macOS entspricht das `traceroute -n api.anthropic.com` (`-n` unterdrückt die DNS-Auflösung der Hops). In der Messung lagen die letzten Hops vor `160.79.104.10` im Adressbereich `162.158.0.0/15`, der Cloudflare gehört. Danach endet die Spur: Die Verbindung wird am Edge-Knoten terminiert, die Weiterleitung ins Backend läuft in einer neuen, internen Verbindung, die von aussen nicht sichtbar ist.

Wem das Präfix gehört und über welches Netz es angekündigt wird, zeigt RIPEstat:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

Im zweiten Ergebnis enden alle AS-Pfade auf `13335 399358`: Cloudflare ist der einzige Upstream des Anthropic-Präfixes.

Eine öffentliche Messung, die den Weg bis in ein Inferenz-Rechenzentrum zurückverfolgt, gibt es nach meiner Recherche nicht, und wegen der Terminierung am Edge ist sie mit Netzwerkwerkzeugen auch nicht möglich. Messdienste wie llmlatency.dev erfassen nur die Antwortzeit bis zum ersten Byte am Edge (Stand September 2026: 96 ms Median aus US-Central, 199 ms aus Deutschland). Die Zeit bis zum ersten Token (Time to First Token) liesse sich zwar aus mehreren Standorten vergleichen, sie hängt aber stärker von Modell, Promptlänge und Auslastung ab als von der Distanz. Ein Rückschluss auf einen Standort ist damit nicht belastbar.

## Was über Anthropics Rechenzentren bekannt ist

Anthropic betreibt bisher kein eigenes, öffentlich zugeordnetes Inferenz-Rechenzentrum mit Adresse. Die Rechenleistung stammt aus Partnerschaften, die Tim Cadenbach im TCDEV-Blog zusammengetragen hat:

| Standort / Partner | Eckdaten | Bekannte Rolle |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Eröffnet Oktober 2025, rund 11 Mrd. USD, etwa 500 000 Trainium2-Chips, über 2,2 GW geplant | Anthropic ist Hauptmieter; vor allem Training |
| Google Cloud | Vereinbarung Oktober 2025 über bis zu 1 Mio. TPUs, über 1 GW ab 2026 | Regionen nicht veröffentlicht |
| Eigene Rechenzentren mit Fluidstack | Angekündigt November 2025, 50 Mrd. USD, Standorte in Texas und New York, Inbetriebnahme ab 2026 | Noch im Aufbau |

Training und Inferenz finden nicht zwingend am selben Ort statt. Die Inferenz kann in beliebigen AWS- oder Google-Cloud-Regionen laufen, und welcher Standort die Antwort für einen Nutzer in der Schweiz rechnet, legt Anthropic nicht offen. Gesichert ist nur, was die Datenschutzerklärung festhält: Vertragspartner und Verantwortlicher für Kunden aus dem EWR, dem Vereinigten Königreich und der Schweiz ist Anthropic Ireland, Limited in Dublin; die Daten werden auf Server in den USA oder in andere Länder ausserhalb des EWR übermittelt, gestützt auf Standardvertragsklauseln.

## Die Steuerung bei Anthropic direkt: nur USA oder global

Die Claude API kennt seit den Modellen der Generation 4.6 den Parameter `inference_geo`. Er hat genau zwei Werte:

| Wert | Wirkung | Preis |
|---|---|---|
| `global` (Standard) | Inferenz in beliebiger verfügbarer Region | Listenpreis |
| `us` | Inferenz ausschliesslich in den USA | Listenpreis × 1,1 |

Einen Wert für die EU oder die Schweiz gibt es nicht. Auch der Speicherort des Workspace (`workspace geo`) lässt sich derzeit nur auf `us` setzen. Die Antwort meldet im Feld `usage.inference_geo`, welche Einstellung angewendet wurde, aber keine konkrete Region. Für Haiku 4.5 und ältere Modelle quittiert die API den Parameter mit einem Fehler 400.

Für claude.ai und Claude Code mit einem Abonnement (Pro, Max, Team, Enterprise) gibt es keine Einstellung des Verarbeitungsorts. Über die direkten Anthropic-Angebote lässt sich die Verarbeitung also weder auf die EU noch auf die Schweiz beschränken. Das geht nur über die Cloud-Anbieter, die Claude in ihren eigenen Regionen betreiben.

## AWS Bedrock: Raum EU plus Schweiz ab Zürich

AWS Bedrock betreibt Claude auf AWS-Infrastruktur. Anthropic hat laut AWS keinen Zugriff auf Prompts, Antworten oder Logs der Kunden. Die Region wählen Sie über die Quellregion Ihres API-Aufrufs und über das Inference Profile:

| Endpunkt-Typ | Präfix der Modell-ID | Verarbeitungsort |
|---|---|---|
| Global Cross-Region Inference | `global.` | Beliebige kommerzielle AWS-Region weltweit |
| Geographic Cross-Region Inference | `eu.` | Nur Regionen innerhalb der Geografie |
| In-Region | ohne Präfix | Nur die aufgerufene Region |

Für die Schweiz ist die Region `eu-central-2` (Zürich) massgebend. Dort sind Opus 5.5, Sonnet 5.5 und Haiku 4.5 verfügbar, allerdings nur als Global- oder EU-Profil und nicht In-Region. Ein Profil, das ausschliesslich in Zürich rechnet, existiert nicht. Wird das EU-Profil aus Zürich aufgerufen, verteilt AWS die Anfragen laut Modellkarte (belegt für Haiku 4.5 und Sonnet 4.6) auf diese Regionen:

| Region | Standort |
|---|---|
| `eu-central-2` | Zürich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Mailand |
| `eu-south-2` | Spanien |
| `eu-west-1` | Irland |
| `eu-west-3` | Paris |

Zürich ist nur dann ein mögliches Ziel, wenn die Anfrage aus Zürich stammt. Die Verarbeitung bleibt damit im Raum EU plus Schweiz. Für Opus 5.5 und Sonnet 5.5 nennt die Modellkarte die Zielregionen nicht; die tatsächliche Liste gibt die API zurück:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `bedrock get-inference-profile` | Liest die Definition eines Inference Profile |
| `--region eu-central-2` | Quellregion Zürich; die Zielliste hängt von der Quellregion ab |
| `--inference-profile-identifier` | ID des EU-Profils für Sonnet 5.5 |
| `--query "models[].modelArn"` | Gibt nur die Modell-ARNs aus; die Region steht jeweils im ARN |

</details>

Gespeichert werden Daten standardmässig nur in der Quellregion. Eine Ausnahme sind Inhalte, die für die Missbrauchserkennung zurückbehalten werden: Sie liegen in der Zielregion. Der Transport zwischen den Regionen läuft verschlüsselt über das AWS-Netz.

### Verarbeitungsregion pro Anfrage belegen

Bedrock ist der einzige der beschriebenen Wege, der die tatsächliche Verarbeitungsregion pro Anfrage protokolliert. CloudTrail schreibt sie in der Quellregion in das Feld `additionalEventData.inferenceRegion`:

```bash
aws cloudtrail lookup-events \
  --region eu-central-2 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 20 \
  --query "Events[].CloudTrailEvent" \
  --output text \
  | jq -r '[.eventTime, .eventName, .additionalEventData.inferenceRegion] | @tsv'
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `cloudtrail lookup-events` | Durchsucht die CloudTrail-Ereignisse der letzten 90 Tage |
| `--region eu-central-2` | Quellregion, in der die Aufrufe protokolliert werden |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filtert auf Ereignisse von Bedrock |
| `--max-results 20` | Begrenzt die Ausgabe auf 20 Ereignisse |
| `--query "Events[].CloudTrailEvent"` | Gibt nur das vollständige Ereignis als JSON-Text aus |
| `--output text` | Ausgabe ohne JSON-Hülle, ein Ereignis pro Zeile |
| `jq -r '… \| @tsv'` | Extrahiert Zeitpunkt, Aktion und Verarbeitungsregion als tabulatorgetrennte Zeile |

</details>

Zusätzlich lassen sich mit einer Service Control Policy (SCP) in AWS Organizations alle Regionen ausserhalb von EU und Schweiz sperren. Ist eine Zielregion eines Profils gesperrt, schlägt die Anfrage fehl, statt auf eine andere Region auszuweichen. Mit dieser Kombination ist der Verarbeitungsort technisch erzwungen und pro Anfrage nachweisbar.

### Claude Code über Bedrock in der EU

Auch Claude Code lässt sich über Bedrock betreiben. In einer `eu-*`-Region wählt Claude Code das Präfix `eu.` automatisch:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Wirkung |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Schaltet Claude Code von der Anthropic API auf Bedrock um |
| `AWS_REGION=eu-central-2` | Quellregion Zürich; daraus folgt das EU-Profil |

Mit `ANTHROPIC_BEDROCK_REGION_PREFIX` lässt sich das Präfix übersteuern. Die AWS-Anmeldung erfolgt über die üblichen Mechanismen (Profil, SSO, Umgebungsvariablen). Abgerechnet wird über das AWS-Konto, nicht über ein Claude-Abonnement.

## Google Vertex AI: nur EU, ausdrücklich ohne Schweiz

Auf Google Cloud (Vertex AI, inzwischen unter dem Namen Gemini Enterprise Agent Platform) ist die Lage für die Schweiz ungünstiger:

| Endpunkt | Modelle (Auswahl) | Verarbeitungsort |
|---|---|---|
| `global` | alle | Beliebige Google-Cloud-Region, ohne Zusicherung |
| Multi-Region `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | Nur EU-Mitgliedstaaten |
| Region `europe-west1` (Belgien) | Haiku 4.5, Sonnet 4.6, Opus 4.6 und ältere | Laut Modellseite Multi-Region Europa |
| Region `europe-west6` (Zürich) | keine | Claude nicht verfügbar |

Google schliesst die Schweiz im EU-Multi-Region-Endpunkt ausdrücklich aus: Er deckt nur EU-Mitgliedstaaten ab, das Vereinigte Königreich und die Schweiz gehören nicht dazu. Mehrere Schweizer Ratgeberseiten nennen `europe-west6` als Weg zu Claude in Zürich; laut der offiziellen Standorttabelle von Google ist dort kein Claude-Modell verfügbar. Ein Protokollfeld mit der tatsächlichen Verarbeitungsregion pro Anfrage dokumentiert Google nicht. Für den globalen Endpunkt hält Google fest, dass sich die Region der Verarbeitung weder steuern noch ermitteln lässt.

Vertex AI eignet sich damit für eine Verarbeitung ausschliesslich in der EU, nicht für eine Verarbeitung in der Schweiz.

## Microsoft Foundry: derzeit keine EU-Option

Microsoft bietet Claude in Foundry in zwei Varianten an. „Hosted on Azure" rechnet auf Azure-Infrastruktur, als Global Standard oder als Data Zone Standard; die Data Zone gibt es für Claude nur für die USA. „Hosted on Anthropic" rechnet auf Anthropic-Infrastruktur, und Microsoft weist darauf hin, dass Daten ausserhalb von Azure und ausserhalb der gewählten Region verarbeitet werden können. Anthropic führt Foundry für Europa als „Coming soon". In Microsoft 365 Copilot und Copilot Studio sind die Anthropic-Modelle von der EU Data Boundary ausgenommen und in EU, EFTA (also auch in der Schweiz) und im Vereinigten Königreich standardmässig deaktiviert.

## Übersicht: welcher Weg welchen Verarbeitungsort zusichert

| Weg | Nur Schweiz | Nur EU | Raum EU plus Schweiz | Region pro Anfrage nachweisbar |
|---|---|---|---|---|
| claude.ai, Claude Code (Abo) | nein | nein | nein | nein |
| Claude API mit `inference_geo` | nein | nein | nein (nur `us`) | nur `us`/`global` |
| AWS Bedrock, Quelle `eu-central-2`, Profil `eu.` | nein | nein | ja | ja (CloudTrail) |
| AWS Bedrock, Quelle in der EU, Profil `eu.` | nein | ja | ja | ja (CloudTrail) |
| Google Vertex AI, Endpunkt `eu` | nein | ja | (nur EU-Teil) | nein |
| Microsoft Foundry | nein | nein | nein | nein |

Ein Weg, der die Verarbeitung von Claude garantiert auf die Schweiz beschränkt, existiert derzeit bei keinem Anbieter. Am nächsten kommt AWS Bedrock mit Quellregion Zürich und EU-Profil. Muss zusätzlich ausgeschlossen sein, dass Daten die EU verlassen (etwa wegen vertraglicher Zusagen an EU-Kunden), lässt sich das EU-Profil aus einer EU-Region wie Frankfurt aufrufen; Zürich ist dann kein Ziel mehr.

## Was die regionale Bindung kostet

Alle drei Cloud-Anbieter und Anthropic selbst verrechnen für eine Bindung an eine Region oder Geografie einen Aufschlag von 10 % gegenüber dem globalen Endpunkt. Die Preise richten sich bei Bedrock nach der Quellregion; Zürich, Frankfurt und North Virginia kosten dasselbe. Das Routing zwischen Regionen ist nicht zusätzlich kostenpflichtig.

Listenpreise in USD pro 1 Mio. Token (Input / Output):

| Modell | Anthropic API, `global` | Anthropic API, `us` | Bedrock oder Vertex, global | Bedrock `eu.` oder Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | nicht verfügbar | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

Bei Vertex AI gilt der Preis der EU-Spalte für Haiku 4.5 und Sonnet 4.6 in der Region `europe-west1`. Ein Rechenbeispiel für eine interne Anwendung mit 50 Mio. Input- und 10 Mio. Output-Token pro Monat:

| Modell | Global | EU-gebunden | Mehrkosten pro Monat |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Die regionale Bindung kostet also 10 % mehr als der globale Endpunkt. Gegenüber dem Listenpreis bei Anthropic direkt mit `inference_geo: "us"` ist ein EU-Profil auf Bedrock gleich teuer. Bei der Wahl ins Gewicht fallen eher die Nebenkosten: Ein AWS- oder Google-Cloud-Konto mit Organisationsrichtlinien, Logging und Budgetwarnungen muss aufgebaut und betrieben werden, und die festen Monatspreise der Claude-Abonnemente entfallen zugunsten einer reinen Abrechnung nach Verbrauch. Ob sich der Wechsel rechnet, hängt vom Volumen ab; ein Abonnement hat einen festen Preis pro Person, bietet aber keine Kontrolle über den Verarbeitungsort.

## Aufbewahrung und Training

Neben dem Ort zählt, wie lange Daten gespeichert werden:

- **Anthropic API:** Prompts und Antworten werden nicht zum Training verwendet, solange keine ausdrückliche Zustimmung vorliegt. Zero Data Retention (ZDR) ist auf Anfrage pro Organisation erhältlich und mit `inference_geo` kombinierbar. Inhalte, die die Missbrauchserkennung markiert, bleiben auch mit ZDR bis zu zwei Jahre gespeichert. Für die Modelle Fable und Mythos ist eine Aufbewahrung von 30 Tagen Pflicht.
- **AWS Bedrock:** Inhalte gehen nicht an Anthropic. Die Aufbewahrung ist pro Modell zwischen `none`, `default` und `aws_review` wählbar; Fable verlangt `aws_review` mit bis zu 30 Tagen Speicherung zur Prüfung durch AWS.
- **Google Vertex AI:** Google verwendet die Daten nur mit vorheriger Zustimmung für Training oder Fine-Tuning.

## Quellen

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, Werte `us` und `global`, Workspace geo, unterstützte Modelle, Feld `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Listenpreise der Anthropic API und Faktor 1,1 für `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, Ausnahmen für markierte Inhalte und für Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland als Verantwortlicher für EWR, UK und Schweiz, Übermittlung in die USA auf Basis von Standardvertragsklauseln.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Übersicht, welche Plattform in welcher Region Datenresidenz bietet; Foundry Europa als „Coming soon".

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Endpunkt-Typen je Region, 10 % Aufschlag für regionale Endpunkte.

7.  [AWS: Modellkarte Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Zielregionen des EU-Profils je Quellregion, inklusive Zürich.

8.  [AWS: Modellkarte Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Verfügbarkeit in `eu-central-2` nur als Geo- und Global-Profil.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Datenverbleib in der Geografie, Speicherort, Sperren von Zielregionen per SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): Feld `additionalEventData.inferenceRegion` in CloudTrail, Preis nach Quellregion.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): kein Zugriff der Modellanbieter auf Prompts, Antworten und Logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): Listenpreise für Global- und Geo-Profile, identisch für Zürich, Frankfurt und North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Anleitung von AWS Schweiz zum EU-Profil ab Zürich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Umgebungsvariablen und automatisches Präfix `eu.` in EU-Regionen.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): Standorttabelle der Claude-Modelle, keine Verfügbarkeit in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU-Multi-Region nur für EU-Mitgliedstaaten, Schweiz und UK ausgeschlossen.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): Listenpreise global, EU-Multi-Region und `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure und Hosted on Anthropic, Data Zone nur USA.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Ausnahme von der EU Data Boundary, Standardeinstellung in EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbachs Übersicht zu Project Rainier, dem Google-TPU-Vertrag und den eigenen Rechenzentren mit Fluidstack.

21.  [Anthropic: Investition von 50 Mrd. USD in US-Infrastruktur](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Ankündigung der eigenen Rechenzentren in Texas und New York mit Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Eingehender Adressbereich `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Präfix-Übersicht und BGP-Status für `160.79.104.0/23` (AS399358, Upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Antwortzeiten am Edge nach Messstandort.

25.  [Fedlex: Datenschutzverordnung (DSV), Anhang 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Liste der Staaten mit angemessenem Datenschutz, inklusive EU/EWR und USA für zertifizierte Unternehmen.
