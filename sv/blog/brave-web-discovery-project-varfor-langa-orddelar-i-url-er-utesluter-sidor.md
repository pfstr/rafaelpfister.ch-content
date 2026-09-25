---
title: "Brave Web Discovery Project: Varför långa orddelar i URL:er utesluter sidor"
navTitle: "Brave: långa URL-delar"
description: "Brave förkastar i Web Discovery Project varje URL vars sökväg innehåller en orddel med fler än 18 tecken. Tyska sammansättningar som Verschlüsselungsgateway omfattas av detta. Eftersom Claudes webbsökning bygger på Braves index påverkar det även synligheten i Claude. Regeln i källkoden, fler heuristiker, canonical-URL:ens roll, ett kontrollskript för den egna sitemapen, en mätning med 825 begrepp på 24 språk och en bedömning av varför regeln missgynnar språk med sammansättningar."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min läsning"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-varfor-langa-orddelar-i-url-er-utesluter-sidor"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/sv/blog/brave-web-discovery-project-varfor-langa-orddelar-i-url-er-utesluter-sidor
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T09:00:21.279Z
translationReview: required
---

# Brave Web Discovery Project: Varför långa orddelar i URL:er utesluter sidor

Claudes webbsökning hämtar sina träffar från indexet i Brave Search. Anthropic har angett Brave Search som underleverantör sedan mars 2025, och citeringarna i Claude-svar överensstämmer i stor utsträckning med Brave-resultaten. Den som vill citeras i Claude-svar måste därför finnas i Braves index. Inskick via Google Search Console, Bing Webmaster Tools eller IndexNow når inte Brave direkt.

Brave fyller sitt index från två källor: en egen crawler och Web Discovery Project (WDP). WDP är en opt-in-funktion i Brave-webbläsaren som anonymt rapporterar besökta sidor till Brave. Innan webbläsaren rapporterar en sida kontrollerar den URL:en med en rad heuristiker. En av dem påverkar särskilt tyskspråkiga webbplatser: Om sökvägen innehåller en orddel med fler än 18 tecken rapporteras sidan aldrig.

## Vad Web Discovery Project rapporterar

WDP samlar in två typer av data: sökfrågor på Google, Bing, Brave och några ytterligare sökmotorer tillsammans med träfflistan, samt sidvisningar med URL, titel, vistelsetid och interaktion. Det finns inga användar-ID:n; varje rapport skickas separat, krypterat och vid slumpmässigt fördelade tidpunkter till Brave.

För att inget privat innehåll ska rapporteras klassar webbläsaren en sida som privat om

- en andra hämtning utan cookies ger en tydligt annorlunda sida (inloggning, personalisering),
- domänen pekar på en privat IP-adress,
- sidan har `noindex` ,
- URL:en ser ut som en hemlig länk (capability-URL, till exempel en delningslänk med token).

Den sista kontrollen utförs av funktionen `dropLongURL`. En URL som klassats som privat sparar webbläsaren permanent i ett Bloom-filter och kontrollerar den inte igen.

## Regeln: ingen orddel över 18 tecken

`dropLongURL` delar upp sökvägen vid snedstreck, punkt, understreck, mellanslag, bindestreck, kolon, plus och semikolon och jämför varje del med gränsvärdet `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` är satt till `18` i källkoden. `return true` betyder: URL:en betraktas som misstänkt. Den avkodade URL:en kontrolleras: Webbläsaren procentkodar icke-ASCII-tecken (`%C3%BC`), WDP omvandlar dem först tillbaka med `cleanCurrentUrl` (`decodeURIComponent`). JavaScript-tecken räknas, alltså UTF-16-kodenheter: Ett `ü` eller ett kinesiskt tecken räknas enkelt, medan ett tecken utanför Unicode-basplanet eller en accent som lagts till som eget tecken räknas dubbelt. Regeln gäller i båda kontrollägena, det normala och det strikta. Brave beskriver själva heuristikerna i README som konservativa: Många offentliga sidor klassas felaktigt som möjliga hemliga länkar, vilket accepteras för WDP:s syfte.

Bindestrecket avgränsar, ett sammanskrivet ord gör det inte. Det avgörande är alltså längden på det längsta enskilda ordet i sluggen, inte längden på hela sluggen:

| Slug | Längsta del | Resultat |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | rapporteras |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | förkastas |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | förkastas |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | rapporteras |

## Varför tyska sluggar påverkas särskilt

Engelska facktermer skrivs som separata ord, tyska skrivs sammansatt. Därtill kommer omskrivningen av omljud: `ü` blir `ue`, vilket gör varje ord med omljud längre. `Zertifikatserneuerung` har 21 tecken, `Verschlüsselungsgateway` har 24 i ASCII-omskrivning. Det är liknande i skandinaviska språk: Svenska och norska skriver också sammansättningar ihop.

På denna webbplats berörs 24 av 750 URL:er enligt en analys av sitemapen: tre tyska artiklar, en engelsk och 20 svenska eller norska översättningar. Det engelska fallet kommer från header-fältet `MessageDirectionality`, som togs över som ord i sluggen.

## När canonical-URL:en hjälper

Om den anropade URL:en misslyckas vid `dropLongURL` kontrollerar webbläsaren sidans canonical-URL. Om den klarar kontrollen rapporterar den sidan under canonical-URL:en. Det täcker det typiska fallet där en sida anropas med spårningsparametrar men har en ren canonical-URL.

Vid en för lång orddel i sluggen hjälper det inte: Canonical-URL:en är vanligen identisk med den anropade URL:en och misslyckas med samma regel. Om båda misslyckas förkastar webbläsaren sidan.

Huruvida det strikta kontrolläget används beror också på canonical-URL:en: Det lättas endast om canonical-URL:en skiljer sig från den anropade URL:en och är kortare. För en sida som anger sig själv som canonical gäller det strikta läget.

## Fler heuristiker i dropLongURL

Utöver längden på orddelarna förkastar `dropLongURL` en URL bland annat vid

- en query string med fler än 30 tecken eller fler än fyra parametrar (i strikt läge från 23 tecken eller två parametrar),
- en sifferföljd med fler än 12 siffror i sökvägen eller query string (i strikt läge fler än 8); specialtecken tas först bort, så en datumstig som `/2026/09/25/` räknas som ett åttasiffrigt tal,
- orddelar som en Markov-klassificerare bedömer som hash,
- sökvägssegment som `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` eller `/token`,
- en e-postadress i URL:en.

För en vanlig artikel-URL utan parametrar är oftast bara ordlängden relevant av detta.

## Bedömning: kvorum och crawler

Regeln påverkar endast vägen via WDP. Två punkter begränsar dess betydelse:

- **Kvorum:** Brave kan avkryptera en sidrapport först när fler än ett visst antal användare har rapporterat samma URL inom 30 dagar från olika nätverk. README anger inte tröskelvärdet. För sidor med få Brave-besökare är WDP därför ändå sällan relevant.
- **Crawler:** Braves crawler arbetar oberoende av WDP. Den uppträder utan egen User-Agent och följer robots.txt-reglerna för Googlebot. En URL med en lång orddel kan ändå komma in i indexet via crawlern.

En lång orddel tar därmed bort en av sidans två vägar in i Braves index; den förblir åtkomlig via crawlern.

## Kontrollera egna URL:er

Följande kommandon läser en sitemap och visar varje URL vars sökväg innehåller en del med fler än 18 tecken. För ett sitemap-index ska de enskilda sitemaparna (`sitemap-0.xml` osv.) kontrolleras efter varandra.

I Linux eller macOS:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `curl -s` | Hämtar sitemapen utan förloppsindikator. |
| `grep -oE '<loc>[^<]+'` | Visar endast `<loc>`-poster (`-o`) med utökade reguljära uttryck (`-E`). |
| `sed -E 's#…##'` | Tar bort `<loc>`, schema och värdnamn; endast sökvägen återstår. |
| `awk -F'[/._ :+;-]'` | Delar upp sökvägen vid samma avgränsare som `dropLongURL`. |
| `length($i) > 18` | Visar längd, orddel och sökväg så snart en del överskrider gränsvärdet. |

</details>

I Windows med PowerShell:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Invoke-RestMethod -Uri` | Hämtar sitemapen och returnerar den direkt som XML-objekt. |
| `$sitemap.urlset.url.loc` | Läser alla `<loc>`-poster från XML:en. |
| `([uri]$loc).AbsolutePath` | Tar bort schema och värdnamn och returnerar sökvägen. |
| `[uri]::UnescapeDataString()` | Avkodar procentkodningen, på samma sätt som WDP gör före kontrollen. |
| `-split '[/._ :+;-]'` | Delar upp sökvägen vid samma avgränsare som `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Behåller endast orddelar med fler än 18 tecken. |

</details>

Båda varianterna kontrollerar endast ordlängden, inte de övriga heuristikerna. Bash-varianten räknar den procentkodade formen och ger därför korrekta värden endast för rena ASCII-sluggar; använd PowerShell-varianten för sluggar med omljud eller andra skriftsystem.

## Anpassa sluggar

För nya artiklar kan regeln följas när sluggen fastställs: Dela sammansättningar i sluggen med bindestreck (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) och förkorta beteckningar som övertagits från facktermer.

För befintliga URL:er bör en ändring vägas mot konsekvenserna. Varje slug-ändring behöver en 301-omdirigering från den gamla till den nya URL:en, en uppdaterad sitemap och anpassade interna länkar. För sidor som redan rankar bra eller länkas till externt ger ändringen liten nytta: De finns ändå i indexet via crawlern. Den är främst meningsfull för nya sidor som ännu inte finns i något index.

## Mätning: Hur starkt påverkas enskilda språk?

Om regeln påverkar språk olika kan mätas genom att köra samma begrepp på olika språk genom Braves originalkod. Skript, data och enskilda resultat finns i det offentliga repositoryt [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); mätningen kan reproduceras där med fem kommandon.

**Upplägg:**

- **Kod:** Braves repository `web-discovery-project`, commit `58b1b53` från den 9 september 2026. Funktionerna `cleanCurrentUrl` och `dropLongURL` körs oförändrade i Node.js; endast kopplingarna till webbläsare och lagring har ersatts. Braves egna testfall för `dropLongURL` ger därmed de förväntade resultaten.
- **Begrepp:** Listan ”Vital Articles Level 3” från engelska Wikipedia, omkring 1 000 centrala artiklar. Via Wikipedias språklänkar hämtades titlarna på samma artiklar på ytterligare 23 språk. 825 begrepp finns på alla 24 språken. Per begrepp ändras därmed endast språket.
- **URL:er:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, skapade som i webbläsaren (procentkodade), sedan som i WDP först genom `cleanCurrentUrl` och därefter genom `dropLongURL` i normalt och strikt läge.
- **Orsak:** Varje förkastad URL kontrollerades en andra gång med längdregeln avstängd. Om den då rapporteras beror förkastandet på längdregeln.
- **Statistik:** 95-procentigt konfidensintervall enligt Wilson; jämförelse med engelska via de parade begreppen (exakt McNemar-test).

Kärnan i analysen:

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

**Resultat** (normalt läge, 825 begrepp per språk):

| Språk | Förkastade | 95 % KI | därav längdregel | p jämfört med engelska |
|---|---|---|---|---|
| Engelska | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Tyska | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Ungerska | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Nederländska | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finska | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Svenska | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Norska | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Danska, franska, spanska, portugisiska, turkiska, polska | vardera 1 (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Ryska, ukrainska, japanska | vardera 1 (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italienska, grekiska, arabiska, hebreiska, persiska, hindi, kinesiska, koreanska | 0 (0,0 %) | 0,0–0,5 % | – | – |

Exempel på förkastade begrepp är `Schwangerschaftsabbruch`, `Empfängnisverhütung` och `Ingenieurwissenschaften` (tyska), `Terhességmegszakítás` (ungerska), `Milieuverontreiniging` (nederländska) och `Tietojenkäsittelytiede` (finska). De enskilda fallen på de övriga språken gäller nästan alla samma begrepp: deoxiribonukleinsyra på respektive nationellt språk. I inget fall rapporterades ett begrepp på det nationella språket medan det förkastades på engelska.

**Analys:**

- Språk med icke-latinsk skrift missgynnas inte. Brave avkodar URL:en före kontrollen, så ett kyrilliskt eller kinesiskt tecken räknas som ett tecken.
- Språk som skriver sammansättningar ihop har en liten men systematisk nackdel. För tyskan är den med p = 0,002 signifikant även när man tar hänsyn till att 23 språk jämförs samtidigt (Bonferroni-tröskel 0,0022). Ungerskan ligger strax över.
- Den uppmätta andelen gäller Wikipedia-titlar, som oftast består av ett eller två ord. Bloggsluggar innehåller fler facktermer; på denna webbplats berörs 3 av 66 tyska artikel-URL:er (4,5 %).

**Begränsningar:** Det som mättes var kontrollregeln, inte det faktiska upptagandet i Braves index. Wikipedia själv finns sannolikt på Braves allowlist och påverkas inte i praktiken; mätningen visar hur regeln behandlar en webbplats utan undantag med sådana URL:er. Den offentliga committen behöver inte motsvara den version som levereras i webbläsaren. Mätningen återger inte heller reservvägen via canonical-URL:en.

## Åsikt: En längdgräns som missgynnar språk med sammansättningar

*Detta avsnitt återger författarens bedömning. Fakta finns i avsnitten ovan.*

Gränsen på 18 tecken räknar tecken och drabbar därmed språk som skriver begrepp sammansatt. Mätningen visar effekten: För identiska begrepp förkastar regeln 1,2 % av de tyska och 0 % av de engelska URL:erna, och vart och ett av dessa tyska förkastanden beror på längdregeln. Den engelska sluggen `data-protection-regulation` klarar kontrollen, `datenschutzgrundverordnung` gör det inte. Effekten är liten, men drabbar alltid samma språk: tyska, ungerska, nederländska, finska och de skandinaviska språken. Enligt min uppfattning är det ett missgynnande av dessa språk, även om det inte är avsiktligt.

### Vem som bär kostnaden

Brave skriver i README att felklassificeringar inte är något stort problem för deras eget syfte. För Brave stämmer det: En förkastad sida kostar Brave en rapport. För de berörda webbplatserna är det en systematisk nackdel som framför allt drabbar facktexter, eftersom just facktermer ofta bildar långa sammansättningar. Eftersom Claudes webbsökning bygger på Braves index saknar dessa sidor en väg in i det index som Claude citerar från.

Därtill finns en serverbaserad allowlist: URL-mönster som Brave lägger till där (`allowlisted`) hoppar över kontrollen. Vilka mönster det gäller är inte offentligt dokumenterat. Enskilda fackwebbplatser kan inte påverka det.

### Vad som talar för regeln

Syftet är berättigat. Delningslänkar med token, exempelvis för delade dokument, utgör en verklig risk, och en sådan länk i sökindexet vore en allvarlig integritetsincident. En hård längdgräns är enkel, snabb och svår att kringgå. Den kan inte heller enkelt ersättas med hash-detektering: I ett tilläggstest med 2 000 slumpmässiga tokens med små bokstäver och 22 tecken klassade Braves `isHash` endast 58 % som hash, medan andelen var 94 % för tokens med små bokstäver och siffror. Alla testade sammansättningar som `schwangerschaftsabbruch` eller `datenschutzgrundverordnung` identifierade funktionen korrekt som inte hash. För hemliga länkar med enbart små bokstäver är längdgränsen därmed det egentliga skyddet. Dessutom når crawlern fortfarande berörda sidor, och den uppmätta andelen är liten.

### Vad Brave skulle kunna ändra

- **Skilj ord från tokens:** En enkel höjning av gränsen för ord som bara består av bokstäver skulle enligt tilläggstestet släppa igenom hemliga länkar med små bokstäver. En ytterligare rimlighetskontroll enbart för orddelar mellan 19 och ungefär 30 tecken vore tänkbar, exempelvis via följden av vokaler och konsonanter eller en liten språkmodell tränad på ord från de berörda språken. Om det är tillräckligt tillförlitligt skulle Brave behöva kontrollera.
- **Dokumentera regeln:** Hjälpsidan om crawlern nämner WDP, men inte heuristikerna. En anvisning till webbplatsägare skulle räcka för att de ska kunna anpassa sina sluggar efter den.

Jag har rapporterat mätningen och denna målkonflikt till Brave som [Issue #498](https://github.com/brave/web-discovery-project/issues/498) i WDP:s repository. Fram till dess återstår bara anpassning på webbplatsens sida: Dela sammansättningar i sluggen med bindestreck. Att detta arbete ligger hos operatörer i vissa språkområden och inte hos metoden är kärnan i min kritik.

*Rättelse från 25.09.2026: En tidigare version föreslog att gränsen för ord med enbart bokstäver skulle höjas generellt; tilläggstestet av hash-detekteringen visar att det skulle försvaga skyddet. En tidigare version av detta avsnitt hävdade att icke-latinska skriftsystem och diakritiska tecken skulle missgynnas särskilt av procentkodningen, baserat på en mätning med procentkodade URL:er. En oberoende efterkontroll visade att Brave avkodar URL:er med `cleanCurrentUrl` före kontrollen. Uppgiften var felaktig och har tagits bort; mätningen ovan använder det korrekta flödet.*

## Källor

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): Braves beskrivning av WDP, med meddelandetyper, andra hämtning utan cookies, capability-URL-heuristiker och kvorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): Källkod med `dropLongURL`, `calculateStrictness` och gränsvärdena `rel_part_len: 18` och `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): Officiell hjälpsida om crawlern, avsaknaden av en egen User-Agent och WDP:s roll.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): Införandet av Brave Search i Anthropics lista över underleverantörer i mars 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): Rapport med ytterligare indikationer, exempelvis parametern `BraveSearchParams` i Claudes webbsökning.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): Avkodning av URL:en före kontrollen; kommentaren i onLocationChange (rad 1724) beskriver den avkodade URL:en som WDP:s interna representation.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): Begreppslista för mätningen; titlarna på de övriga språken kommer från Wikipedias API:s språklänkar.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): Mätskript, dataset och enskilda resultat från språkmätningen i denna artikel, inklusive tilläggstestet för hash-detektering (`hash-test.mjs`) och den första version som markerats som felaktig.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): Återkoppling till Brave med mätresultaten och målkonflikten mellan längdgräns och hash-detektering.
