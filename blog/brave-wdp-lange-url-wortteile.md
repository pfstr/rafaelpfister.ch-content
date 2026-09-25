---
title: "Brave Web Discovery Project: Warum lange Wortteile in URLs Seiten ausschliessen"
navTitle: "Brave: lange URL-Teile"
description: "Brave verwirft im Web Discovery Project jede URL, deren Pfad einen Wortteil mit mehr als 18 Zeichen enthält. Deutsche Komposita wie Verschlüsselungsgateway fallen darunter. Weil Claudes Websuche auf dem Brave-Index aufsetzt, betrifft das auch die Sichtbarkeit in Claude. Die Regel im Quellcode, weitere Heuristiken, die Rolle des Canonical, ein Prüfskript für die eigene Sitemap eine Messung mit 825 Begriffen in 24 Sprachen und eine Einschätzung, warum die Regel Komposita-Sprachen benachteiligt."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 Min. Lesezeit"
themen:
  - "claude"
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-wdp-lange-url-wortteile"
translationId: "article-2cf5ffd0d1886c78"
url: "https://rafaelpfister.ch/blog/brave-wdp-lange-url-wortteile"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
---
# Brave Web Discovery Project: Warum lange Wortteile in URLs Seiten ausschliessen

Claudes Websuche liefert ihre Treffer aus dem Index von Brave Search. Anthropic führt Brave Search seit März 2025 als Unterauftragsverarbeiter, und die Zitate in Claude-Antworten decken sich weitgehend mit den Brave-Ergebnissen. Wer in Claude-Antworten zitiert werden will, muss deshalb im Brave-Index stehen. Einreichungen über Google Search Console, Bing Webmaster Tools oder IndexNow erreichen Brave nicht direkt.

Brave füllt seinen Index aus zwei Quellen: einem eigenen Crawler und dem Web Discovery Project (WDP). Das WDP ist eine Opt-in-Funktion im Brave-Browser, die besuchte Seiten anonymisiert an Brave meldet. Bevor der Browser eine Seite meldet, prüft er die URL mit einer Reihe von Heuristiken. Eine davon betrifft deutschsprachige Websites besonders: Enthält der Pfad einen Wortteil mit mehr als 18 Zeichen, wird die Seite nie gemeldet.

## Was das Web Discovery Project meldet

Das WDP sammelt zwei Arten von Daten: Suchanfragen auf Google, Bing, Brave und einigen weiteren Suchmaschinen samt Trefferliste, und Seitenaufrufe mit URL, Titel, Verweildauer und Interaktion. Nutzer-IDs gibt es keine; jede Meldung geht einzeln, verschlüsselt und zeitlich zufällig verteilt an Brave.

Damit keine privaten Inhalte gemeldet werden, stuft der Browser eine Seite als privat ein, wenn

- ein zweiter Abruf ohne Cookies eine deutlich andere Seite liefert (Login, Personalisierung),
- die Domain auf eine private IP-Adresse zeigt,
- die Seite `noindex` trägt,
- die URL wie ein Geheim-Link aussieht (Capability-URL, etwa ein Freigabelink mit Token).

Die letzte Prüfung übernimmt die Funktion `dropLongURL`. Eine als privat eingestufte URL speichert der Browser dauerhaft in einem Bloom-Filter und prüft sie nicht erneut.

## Die Regel: kein Wortteil über 18 Zeichen

`dropLongURL` zerlegt den Pfad an Schrägstrich, Punkt, Unterstrich, Leerzeichen, Bindestrich, Doppelpunkt, Plus und Semikolon und vergleicht jeden Teil mit dem Grenzwert `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` ist im Quellcode auf `18` gesetzt. `return true` heisst: Die URL gilt als verdächtig. Geprüft wird die dekodierte URL: Der Browser führt Nicht-ASCII-Zeichen prozentkodiert (`%C3%BC`), das WDP wandelt sie vorher mit `cleanCurrentUrl` (`decodeURIComponent`) zurück. Gezählt werden JavaScript-Zeichen, also UTF-16-Codeeinheiten: Ein `ü` oder ein chinesisches Schriftzeichen zählt einfach, ein Zeichen ausserhalb der Unicode-Basisebene oder ein als eigenes Zeichen angehängter Akzent doppelt. Die Regel gilt in beiden Prüfmodi, dem normalen und dem strengen. Brave bezeichnet die Heuristiken im README selbst als konservativ: Viele öffentliche Seiten werden fälschlich als möglicher Geheim-Link eingestuft, was für den Zweck des WDP in Kauf genommen wird.

Der Bindestrich trennt, ein zusammengeschriebenes Wort nicht. Entscheidend ist also die Länge des längsten Einzelworts im Slug, nicht die Länge des ganzen Slugs:

| Slug | Längster Teil | Ergebnis |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | wird gemeldet |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | verworfen |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | verworfen |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | wird gemeldet |

## Warum deutsche Slugs besonders betroffen sind

Englische Fachbegriffe werden getrennt geschrieben, deutsche zusammengesetzt. Dazu kommt die Umschrift von Umlauten: Aus `ü` wird `ue`, jedes Umlautwort wird dadurch länger. `Zertifikatserneuerung` hat 21 Zeichen, `Verschlüsselungsgateway` in ASCII-Umschrift 24. In skandinavischen Sprachen ist es ähnlich: Schwedisch und Norwegisch setzen Komposita ebenfalls zusammen.

Auf dieser Website sind nach einer Auswertung der Sitemap 24 von 750 URLs betroffen: drei deutsche Artikel, ein englischer und 20 schwedische oder norwegische Übersetzungen. Der englische Fall stammt aus dem Header-Feld `MessageDirectionality`, das als Wort in den Slug übernommen wurde.

## Wann der Canonical hilft

Scheitert die aufgerufene URL an `dropLongURL`, prüft der Browser die Canonical-URL der Seite. Besteht diese die Prüfung, meldet er die Seite unter der Canonical-URL. Das deckt den typischen Fall ab, dass eine Seite mit Tracking-Parametern aufgerufen wird, aber einen sauberen Canonical hat.

Bei einem zu langen Wortteil im Slug hilft das nicht: Die Canonical-URL ist in der Regel identisch mit der aufgerufenen URL und scheitert an derselben Regel. Scheitern beide, verwirft der Browser die Seite.

Ob der strenge Prüfmodus greift, hängt ebenfalls am Canonical: Er wird nur gelockert, wenn der Canonical von der aufgerufenen URL abweicht und kürzer ist. Bei einer Seite, die sich selbst als Canonical angibt, gilt der strenge Modus.

## Weitere Heuristiken in dropLongURL

Neben der Länge der Wortteile verwirft `dropLongURL` eine URL unter anderem bei

- einem Query-String mit mehr als 30 Zeichen oder mehr als vier Parametern (im strengen Modus ab 23 Zeichen oder zwei Parametern),
- einer Ziffernfolge mit mehr als 12 Ziffern im Pfad oder Query-String (im strengen Modus mehr als 8); Sonderzeichen werden vorher entfernt, ein Datumspfad wie `/2026/09/25/` zählt also als achtstellige Zahl,
- Wortteilen, die ein Markov-Klassifikator als Hash einstuft,
- Pfadsegmenten wie `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` oder `/token`,
- einer E-Mail-Adresse in der URL.

Für eine gewöhnliche Artikel-URL ohne Parameter ist davon meist nur die Wortlänge relevant.

## Einordnung: Quorum und Crawler

Die Regel betrifft nur den Weg über das WDP. Zwei Punkte begrenzen ihre Bedeutung:

- **Quorum:** Brave kann eine Seitenmeldung erst entschlüsseln, wenn mehr als eine bestimmte Anzahl Nutzer dieselbe URL innerhalb von 30 Tagen aus verschiedenen Netzen gemeldet hat. Den Schwellenwert nennt das README nicht. Für Seiten mit wenigen Brave-Besuchern greift das WDP deshalb ohnehin kaum.
- **Crawler:** Der Brave-Crawler arbeitet unabhängig vom WDP. Er tritt ohne eigenen User-Agent auf und folgt den robots.txt-Regeln für Googlebot. Eine URL mit langem Wortteil kann über den Crawler trotzdem in den Index gelangen.

Ein langer Wortteil nimmt einer Seite damit einen der beiden Wege in den Brave-Index; über den Crawler bleibt sie erreichbar.

## Eigene URLs prüfen

Die folgenden Befehle lesen eine Sitemap und geben jede URL aus, deren Pfad einen Teil mit mehr als 18 Zeichen enthält. Bei einem Sitemap-Index die einzelnen Sitemaps (`sitemap-0.xml` usw.) nacheinander prüfen.

Unter Linux oder macOS:

```bash
curl -s https://example.com/sitemap-0.xml \
  | grep -oE '<loc>[^<]+' \
  | sed -E 's#<loc>https?://[^/]*##' \
  | awk -F'[/._ :+;-]' '{
      for (i = 1; i <= NF; i++)
        if (length($i) > 18) { print length($i), $i, $0; break }
    }'
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `curl -s` | Lädt die Sitemap ohne Fortschrittsanzeige. |
| `grep -oE '<loc>[^<]+'` | Gibt nur die `<loc>`-Einträge aus (`-o`), mit erweiterten regulären Ausdrücken (`-E`). |
| `sed -E 's#…##'` | Entfernt `<loc>`, Schema und Hostname; übrig bleibt der Pfad. |
| `awk -F'[/._ :+;-]'` | Zerlegt den Pfad an denselben Trennzeichen wie `dropLongURL`. |
| `length($i) > 18` | Gibt Länge, Wortteil und Pfad aus, sobald ein Teil den Grenzwert überschreitet. |

</details>

Unter Windows mit PowerShell:

```powershell
$sitemap = Invoke-RestMethod -Uri 'https://example.com/sitemap-0.xml'
foreach ($loc in $sitemap.urlset.url.loc) {
    $pfad = [uri]::UnescapeDataString(([uri]$loc).AbsolutePath)
    $zuLang = $pfad -split '[/._ :+;-]' |
        Where-Object { $_.Length -gt 18 }
    if ($zuLang) {
        '{0}  ->  {1}' -f ($zuLang -join ', '), $pfad
    }
}
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `Invoke-RestMethod -Uri` | Lädt die Sitemap und gibt sie direkt als XML-Objekt zurück. |
| `$sitemap.urlset.url.loc` | Liest alle `<loc>`-Einträge aus dem XML. |
| `([uri]$loc).AbsolutePath` | Schneidet Schema und Hostname ab und liefert den Pfad. |
| `[uri]::UnescapeDataString()` | Dekodiert die Prozentkodierung, wie es auch das WDP vor der Prüfung tut. |
| `-split '[/._ :+;-]'` | Zerlegt den Pfad an denselben Trennzeichen wie `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Behält nur Wortteile mit mehr als 18 Zeichen. |

</details>

Beide Varianten prüfen nur die Wortlänge, nicht die übrigen Heuristiken. Die Bash-Variante zählt die prozentkodierte Form und liefert deshalb nur für reine ASCII-Slugs korrekte Werte; für Slugs mit Umlauten oder anderen Schriften die PowerShell-Variante verwenden.

## Slugs anpassen

Für neue Artikel lässt sich die Regel beim Festlegen des Slugs einhalten: Komposita im Slug mit Bindestrich trennen (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) und aus Fachbegriffen übernommene Bezeichner kürzen.

Bei bestehenden URLs ist eine Änderung abzuwägen. Jede Slug-Änderung braucht eine 301-Weiterleitung von der alten auf die neue URL, eine aktualisierte Sitemap und angepasste interne Links. Bei Seiten, die bereits gut ranken oder von aussen verlinkt sind, bringt die Umstellung wenig: Sie sind über den Crawler ohnehin im Index. Sinnvoll ist sie vor allem bei jungen Seiten, die noch in keinem Index stehen.

## Messung: Wie stark sind einzelne Sprachen betroffen?

Ob die Regel Sprachen unterschiedlich trifft, lässt sich messen, indem man dieselben Begriffe in verschiedenen Sprachen durch Braves Originalcode schickt. Skripte, Daten und Einzelergebnisse liegen im öffentlichen Repository [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); die Messung lässt sich dort mit fünf Befehlen reproduzieren.

**Aufbau:**

- **Code:** Braves Repository `web-discovery-project`, Commit `58b1b53` vom 9. September 2026. Die Funktionen `cleanCurrentUrl` und `dropLongURL` laufen unverändert in Node.js; ersetzt sind nur die Anbindungen an Browser und Speicher. Braves eigene Testfälle für `dropLongURL` liefern damit die erwarteten Ergebnisse.
- **Begriffe:** die Liste „Vital Articles Level 3" der englischen Wikipedia, rund 1000 zentrale Artikel. Über die Sprachverknüpfungen der Wikipedia wurden die Titel derselben Artikel in 23 weiteren Sprachen abgerufen. 825 Begriffe existieren in allen 24 Sprachen. Pro Begriff ändert sich damit nur die Sprache.
- **URLs:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, gebildet wie im Browser (prozentkodiert), dann wie im WDP zuerst durch `cleanCurrentUrl` und anschliessend durch `dropLongURL` im normalen und im strengen Modus.
- **Ursache:** Jede verworfene URL wurde ein zweites Mal geprüft, mit abgeschalteter Längenregel. Wird sie dann gemeldet, geht die Verwerfung auf die Längenregel zurück.
- **Statistik:** 95-%-Konfidenzintervall nach Wilson; Vergleich mit Englisch über die gepaarten Begriffe (exakter McNemar-Test).

Der Kern der Auswertung:

```javascript
for (const begriff of begriffe) {
  const kodiert = new URL(wikiUrl(sprache, begriff)).href;
  const url = WDP.cleanCurrentUrl(kodiert);
  const verworfen = WDP.dropLongURL(url);
  WDP.rel_part_len = Infinity;   // Längenregel aus
  const ohneLaenge = WDP.dropLongURL(url);
  WDP.rel_part_len = 18;         // Originalwert
  if (verworfen && !ohneLaenge) durchLaengenregel++;
}
```

**Ergebnis** (normaler Modus, 825 Begriffe je Sprache):

| Sprache | Verworfen | 95-%-KI | davon Längenregel | p gegenüber Englisch |
|---|---|---|---|---|
| Englisch | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Deutsch | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Ungarisch | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Niederländisch | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finnisch | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Schwedisch | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Norwegisch | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Dänisch, Französisch, Spanisch, Portugiesisch, Türkisch, Polnisch | je 1 (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Russisch, Ukrainisch, Japanisch | je 1 (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italienisch, Griechisch, Arabisch, Hebräisch, Persisch, Hindi, Chinesisch, Koreanisch | 0 (0,0 %) | 0,0–0,5 % | – | – |

Verworfen wurden zum Beispiel `Schwangerschaftsabbruch`, `Empfängnisverhütung` und `Ingenieurwissenschaften` (Deutsch), `Terhességmegszakítás` (Ungarisch), `Milieuverontreiniging` (Niederländisch) und `Tietojenkäsittelytiede` (Finnisch). Die Einzelfälle in den übrigen Sprachen betreffen fast alle denselben Begriff: Desoxyribonukleinsäure in der jeweiligen Landessprache. In keinem Fall wurde ein Begriff in der Landessprache gemeldet und auf Englisch verworfen.

**Auswertung:**

- Sprachen mit nicht-lateinischer Schrift werden nicht benachteiligt. Brave dekodiert die URL vor der Prüfung, ein kyrillisches oder chinesisches Zeichen zählt dadurch als ein Zeichen.
- Sprachen, die Komposita zusammenschreiben, haben einen kleinen, aber systematischen Nachteil. Für Deutsch ist er mit p = 0,002 auch dann signifikant, wenn man berücksichtigt, dass 23 Sprachen gleichzeitig verglichen wurden (Bonferroni-Schwelle 0,0022). Ungarisch liegt knapp darüber.
- Der gemessene Anteil gilt für Wikipedia-Titel, die meist aus einem oder zwei Wörtern bestehen. Blog-Slugs enthalten mehr Fachbegriffe; auf dieser Website sind 3 von 66 deutschen Artikel-URLs betroffen (4,5 %).

**Grenzen:** Gemessen wurde die Prüfregel, nicht die tatsächliche Aufnahme in den Brave-Index. Wikipedia selbst steht vermutlich auf Braves Freigabeliste und ist real nicht betroffen; die Messung zeigt, wie die Regel eine Website ohne Freigabe mit solchen URLs behandelt. Der öffentliche Commit muss nicht der im Browser ausgelieferten Version entsprechen. Den Ausweichpfad über die Canonical-URL bildet die Messung nicht ab.

## Meinung: Eine Längengrenze, die Komposita-Sprachen benachteiligt

*Dieser Abschnitt gibt die Einschätzung des Autors wieder. Die Fakten dazu stehen in den Abschnitten oben.*

Die 18-Zeichen-Grenze zählt Zeichen und trifft damit Sprachen, die Begriffe zusammenschreiben. Die Messung zeigt den Effekt: Bei identischen Begriffen verwirft die Regel 1,2 % der deutschen und 0 % der englischen URLs, und jede dieser deutschen Verwerfungen geht auf die Längenregel zurück. Der englische Slug `data-protection-regulation` besteht die Prüfung, `datenschutzgrundverordnung` nicht. Der Effekt ist klein, trifft aber immer dieselben Sprachen: Deutsch, Ungarisch, Niederländisch, Finnisch und die skandinavischen Sprachen. Aus meiner Sicht ist das eine Benachteiligung dieser Sprachen, auch wenn sie nicht beabsichtigt ist.

### Wer die Kosten trägt

Brave schreibt im README, Fehleinstufungen seien für den eigenen Zweck kein grosses Problem. Für Brave trifft das zu: Eine verworfene Seite kostet Brave eine Meldung. Für die betroffenen Websites ist es ein systematischer Nachteil, der vor allem Fachtexte trifft, weil gerade Fachbegriffe lange Komposita bilden. Weil Claudes Websuche auf dem Brave-Index aufsetzt, fehlt diesen Seiten ein Weg in den Index, aus dem Claude zitiert.

Hinzu kommt eine serverseitige Freigabeliste: URL-Muster, die Brave dort aufnimmt (`allowlisted`), überspringen die Prüfung. Welche Muster das sind, ist nicht öffentlich dokumentiert. Einzelne Fachseiten können darauf keinen Einfluss nehmen.

### Was für die Regel spricht

Der Zweck ist berechtigt. Freigabelinks mit Token, etwa für geteilte Dokumente, sind ein reales Risiko, und ein solcher Link im Suchindex wäre ein ernster Datenschutzvorfall. Eine harte Längengrenze ist einfach, schnell und schwer zu umgehen. Sie ist auch nicht einfach durch die Hash-Erkennung ersetzbar: In einem Zusatztest mit 2000 zufälligen Tokens aus Kleinbuchstaben mit 22 Zeichen stufte Braves `isHash` nur 58 % als Hash ein, bei Tokens aus Kleinbuchstaben und Ziffern 94 %. Alle getesteten Komposita wie `schwangerschaftsabbruch` oder `datenschutzgrundverordnung` erkannte die Funktion korrekt als kein Hash. Für Geheim-Links aus reinen Kleinbuchstaben ist die Längengrenze damit der eigentliche Schutz. Zudem erreicht der Crawler betroffene Seiten weiterhin, und der gemessene Anteil ist klein.

### Was Brave ändern könnte

- **Wörter von Tokens unterscheiden:** Eine einfache Anhebung der Grenze für reine Buchstabenwörter würde nach dem Zusatztest Geheim-Links aus Kleinbuchstaben durchlassen. Denkbar ist eine zusätzliche Plausibilitätsprüfung nur für Wortteile zwischen 19 und etwa 30 Zeichen, etwa über die Abfolge von Vokalen und Konsonanten oder ein kleines Sprachmodell, das auf Wörtern der betroffenen Sprachen trainiert ist. Ob das zuverlässig genug ist, müsste Brave prüfen.
- **Die Regel dokumentieren:** Die Hilfeseite zum Crawler erwähnt das WDP, aber nicht die Heuristiken. Ein Hinweis für Website-Betreiber würde genügen, damit sie ihre Slugs danach ausrichten können.

Die Messung und diesen Zielkonflikt habe ich Brave als [Issue #498](https://github.com/brave/web-discovery-project/issues/498) im Repository des WDP gemeldet. Bis dahin bleibt nur die Anpassung auf Seiten der Website: Komposita im Slug mit Bindestrich trennen. Dass diese Arbeit bei den Betreibern bestimmter Sprachräume liegt und nicht beim Verfahren, ist der Kern meiner Kritik.

*Korrektur vom 25.09.2026: Eine frühere Fassung schlug vor, die Grenze für reine Buchstabenwörter pauschal anzuheben; der Zusatztest zur Hash-Erkennung zeigt, dass das den Schutz schwächen würde. Eine frühere Fassung dieses Abschnitts behauptete, nicht-lateinische Schriften und Diakritika würden durch die Prozentkodierung besonders stark benachteiligt, gestützt auf eine Messung mit prozentkodierten URLs. Eine unabhängige Nachprüfung hat gezeigt, dass Brave URLs vor der Prüfung mit `cleanCurrentUrl` dekodiert. Die Aussage war falsch und ist entfernt; die Messung oben verwendet den korrekten Ablauf.*

## Quellen

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): Beschreibung des WDP durch Brave, mit Nachrichtentypen, Zweitabruf ohne Cookies, Capability-URL-Heuristiken und Quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): Quellcode mit `dropLongURL`, `calculateStrictness` und den Grenzwerten `rel_part_len: 18` und `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): Offizielle Hilfeseite zum Crawler, zum fehlenden eigenen User-Agent und zur Rolle des WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): Eintrag von Brave Search in Anthropics Liste der Unterauftragsverarbeiter im März 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): Bericht mit weiteren Hinweisen, etwa dem Parameter `BraveSearchParams` in Claudes Websuche.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): Dekodierung der URL vor der Prüfung; der Kommentar in onLocationChange (Zeile 1724) beschreibt die dekodierte URL als interne Darstellung des WDP.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): Begriffsliste der Messung; die Titel in den übrigen Sprachen stammen aus den Sprachverknüpfungen der Wikipedia-API.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): Messskripte, Datensatz und Einzelergebnisse der Sprachmessung in diesem Artikel, einschliesslich des Zusatztests zur Hash-Erkennung (`hash-test.mjs`) und der als fehlerhaft markierten ersten Fassung.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): Rückmeldung an Brave mit den Messergebnissen und dem Zielkonflikt zwischen Längengrenze und Hash-Erkennung.
