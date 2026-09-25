---
title: "Brave Web Discovery Project: Warum lange Wortteile in URLs Seiten ausschliessen"
navTitle: "Brave: lange URL-Teile"
description: "Brave verwirft im Web Discovery Project jede URL, deren Pfad einen Wortteil mit mehr als 18 Zeichen enthält. Deutsche Komposita wie Verschlüsselungsgateway fallen darunter. Weil Claudes Websuche auf dem Brave-Index aufsetzt, betrifft das auch die Sichtbarkeit in Claude. Die Regel im Quellcode, weitere Heuristiken, die Rolle des Canonical, ein Prüfskript für die eigene Sitemap und eine Einschätzung, warum die Regel Sprachen ungleich behandelt."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "11 Min. Lesezeit"
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

`rel_part_len` ist im Quellcode auf `18` gesetzt. `return true` heisst: Die URL gilt als verdächtig. Die Regel gilt in beiden Prüfmodi, dem normalen und dem strengen. Brave bezeichnet die Heuristiken im README selbst als konservativ: Viele öffentliche Seiten werden fälschlich als möglicher Geheim-Link eingestuft, was für den Zweck des WDP in Kauf genommen wird.

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
    $pfad = ([uri]$loc).AbsolutePath
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
| `-split '[/._ :+;-]'` | Zerlegt den Pfad an denselben Trennzeichen wie `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Behält nur Wortteile mit mehr als 18 Zeichen. |

</details>

Beide Varianten prüfen nur die Wortlänge, nicht die übrigen Heuristiken.

## Slugs anpassen

Für neue Artikel lässt sich die Regel beim Festlegen des Slugs einhalten: Komposita im Slug mit Bindestrich trennen (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) und aus Fachbegriffen übernommene Bezeichner kürzen.

Bei bestehenden URLs ist eine Änderung abzuwägen. Jede Slug-Änderung braucht eine 301-Weiterleitung von der alten auf die neue URL, eine aktualisierte Sitemap und angepasste interne Links. Bei Seiten, die bereits gut ranken oder von aussen verlinkt sind, bringt die Umstellung wenig: Sie sind über den Crawler ohnehin im Index. Sinnvoll ist sie vor allem bei jungen Seiten, die noch in keinem Index stehen.

## Meinung: Eine Längengrenze, die Sprachen ungleich behandelt

*Dieser Abschnitt gibt die Einschätzung des Autors wieder. Die Fakten dazu stehen in den Abschnitten oben und in den Quellen.*

Die 18-Zeichen-Grenze zählt Zeichen und bewertet damit Sprachen unterschiedlich. Derselbe Inhalt wird gemeldet oder verworfen, je nachdem, in welcher Sprache der Slug geschrieben ist. Aus meiner Sicht ist das eine Benachteiligung nach Sprache, auch wenn sie nicht beabsichtigt ist.

### Komposita

Der englische Slug `certificate-renewal` besteht die Prüfung, sein längster Teil hat 11 Zeichen. Das deutsche `zertifikatserneuerung` mit 21 Zeichen wird verworfen. Beide bezeichnen dasselbe. Betroffen sind alle Sprachen, die zusammengesetzte Wörter zusammenschreiben: Deutsch, Niederländisch, Schwedisch, Norwegisch, Dänisch, Finnisch, Ungarisch. Englisch und die romanischen Sprachen trennen Begriffe mit Leerzeichen, im Slug also mit Bindestrich, und sind kaum betroffen.

### Diakritika und nicht-lateinische Schriften

Noch deutlicher wird die Ungleichbehandlung bei Zeichen ausserhalb von ASCII. Browser führen URLs nach dem URL-Standard prozentkodiert: Jedes Nicht-ASCII-Zeichen wird in zwei bis vier Bytes zerlegt, jedes Byte in drei Zeichen wie `%D0`. Die Funktion `dropLongURL` prüft die kodierte Form. Ein kyrillischer Buchstabe zählt damit sechs Zeichen, ein chinesisches Schriftzeichen neun:

| Wort | Sprache | Buchstaben | Zeichen in der URL | Ergebnis |
|---|---|---|---|---|
| `certificate-renewal` | Englisch | 11 (längster Teil) | 11 | wird gemeldet |
| `zertifikatserneuerung` | Deutsch | 21 | 21 | verworfen |
| `sähköpostipalvelin` (Mailserver) | Finnisch | 18 | 28 | verworfen |
| `levelezőszerver` (Mailserver) | Ungarisch | 15 | 20 | verworfen |
| `почта` (Post) | Russisch | 5 | 30 | verworfen |
| `ελληνικά` (Griechisch) | Griechisch | 8 | 48 | verworfen |
| `بريد` (Post) | Arabisch | 4 | 24 | verworfen |
| `メール` (Mail) | Japanisch | 3 | 27 | verworfen |
| `电子邮件` (E-Mail) | Chinesisch | 4 | 36 | verworfen |

Ein Wort in kyrillischer, griechischer oder arabischer Schrift darf also höchstens drei Buchstaben haben, ein chinesisches oder japanisches höchstens zwei Schriftzeichen. Websites in diesen Sprachen gelangen über das WDP praktisch nur mit lateinisch transliterierten Slugs in den Brave-Index.

### Wer die Kosten trägt

Brave schreibt im README, Fehleinstufungen seien für den eigenen Zweck kein grosses Problem. Für Brave trifft das zu: Eine verworfene Seite kostet Brave eine Meldung. Für die betroffenen Websites ist es ein systematischer Nachteil, der immer dieselben Sprachen trifft. Weil Claudes Websuche auf dem Brave-Index aufsetzt, setzt sich die Ungleichbehandlung in KI-Antworten fort: Inhalte in diesen Sprachen haben einen Weg weniger in den Index, aus dem Claude zitiert.

Hinzu kommt eine serverseitige Freigabeliste: URL-Muster, die Brave dort aufnimmt (`allowlisted`), überspringen die Prüfung. Welche Muster das sind, ist nicht öffentlich dokumentiert. Einzelne Fachseiten können darauf keinen Einfluss nehmen.

### Was für die Regel spricht

Der Zweck ist berechtigt. Freigabelinks mit Token, etwa für geteilte Dokumente, sind ein reales Risiko, und ein solcher Link im Suchindex wäre ein ernster Datenschutzvorfall. Eine harte Längengrenze ist einfach, schnell und schwer zu umgehen. Zudem erreicht der Crawler betroffene Seiten weiterhin.

### Was Brave ändern könnte

- **Nach dem Dekodieren zählen:** Die Grenze auf die dekodierten Zeichen statt auf die Prozentkodierung anzuwenden, würde die Benachteiligung nicht-lateinischer Schriften und von Diakritika weitgehend beheben.
- **Den vorhandenen Klassifikator nutzen:** Der Code enthält bereits einen Markov-Klassifikator, der zufällig aussehende Zeichenketten als Hash erkennt. Ein Token ist meist zufällig, ein langes Wort einer natürlichen Sprache nicht. Für Wortteile, die der Klassifikator als natürliche Sprache einstuft, liesse sich die Grenze anheben. Ob der Klassifikator dafür in allen Sprachen zuverlässig genug ist, müsste Brave prüfen.
- **Die Regel dokumentieren:** Die Hilfeseite zum Crawler erwähnt das WDP, aber nicht die Heuristiken. Ein Hinweis für Website-Betreiber würde genügen, damit sie ihre Slugs danach ausrichten können.

Bis dahin bleibt nur die Anpassung auf Seiten der Website: Komposita trennen, Slugs transliterieren. Dass diese Arbeit bei den Betreibern bestimmter Sprachräume liegt und nicht beim Verfahren, ist der Kern meiner Kritik.

## Quellen

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): Beschreibung des WDP durch Brave, mit Nachrichtentypen, Zweitabruf ohne Cookies, Capability-URL-Heuristiken und Quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): Quellcode mit `dropLongURL`, `calculateStrictness` und den Grenzwerten `rel_part_len: 18` und `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): Offizielle Hilfeseite zum Crawler, zum fehlenden eigenen User-Agent und zur Rolle des WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): Eintrag von Brave Search in Anthropics Liste der Unterauftragsverarbeiter im März 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): Bericht mit weiteren Hinweisen, etwa dem Parameter `BraveSearchParams` in Claudes Websuche.

6.  [WHATWG: URL Standard, Percent-encoded bytes](https://url.spec.whatwg.org/#percent-encoded-bytes): Regeln, nach denen Browser Nicht-ASCII-Zeichen im Pfad als UTF-8-Bytes prozentkodieren.
