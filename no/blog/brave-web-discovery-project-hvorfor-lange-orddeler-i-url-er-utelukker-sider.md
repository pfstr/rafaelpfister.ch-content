---
title: "Brave Web Discovery Project: Hvorfor lange orddeler i URL-er utelukker sider"
navTitle: "Brave: lange URL-deler"
description: "Brave forkaster i Web Discovery Project alle URL-er der stien inneholder en orddel med mer enn 18 tegn. Tyske sammensetninger som Verschlüsselungsgateway rammes av dette. Fordi Claudes nettsøk bygger på Brave-indeksen, påvirker det også synligheten i Claude. Regelen i kildekoden, flere heuristikker, canonicals rolle, et kontrollskript for eget nettstedskart, en måling med 825 begreper på 24 språk og en vurdering av hvorfor regelen diskriminerer språk med sammensetninger."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min lesetid"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-hvorfor-lange-orddeler-i-url-er-utelukker-sider"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/no/blog/brave-web-discovery-project-hvorfor-lange-orddeler-i-url-er-utelukker-sider
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T09:01:35.091Z
translationReview: automatic
---

# Brave Web Discovery Project: Hvorfor lange orddeler i URL-er utelukker sider

Claudes nettsøk leverer treffene sine fra indeksen til Brave Search. Anthropic har oppført Brave Search som underleverandør siden mars 2025, og siteringene i Claude-svar samsvarer i stor grad med Brave-resultatene. De som vil siteres i Claude-svar, må derfor være i Brave-indeksen. Innsendinger via Google Search Console, Bing Webmaster Tools eller IndexNow når ikke Brave direkte.

Brave fyller indeksen sin fra to kilder: en egen crawler og Web Discovery Project (WDP). WDP er en opt-in-funksjon i Brave-nettleseren som anonymt rapporterer besøkte sider til Brave. Før nettleseren rapporterer en side, kontrollerer den URL-en med en rekke heuristikker. Én av dem berører spesielt tyskspråklige nettsteder: Inneholder stien en orddel med mer enn 18 tegn, rapporteres siden aldri.

## Hva Web Discovery Project rapporterer

WDP samler inn to typer data: søk på Google, Bing, Brave og enkelte andre søkemotorer, inkludert resultatlisten, samt sidebesøk med URL, tittel, oppholdstid og interaksjon. Det finnes ingen bruker-ID-er; hver rapport sendes enkeltvis, kryptert og med tilfeldig tidsfordeling til Brave.

For at ikke privat innhold skal rapporteres, klassifiserer nettleseren en side som privat dersom

- en ny henting uten informasjonskapsler leverer en betydelig annen side (innlogging, personalisering),
- domenet peker til en privat IP-adresse,
- siden har `noindex`,
- URL-en ser ut som en hemmelig lenke (Capability-URL, for eksempel en delingslenke med token).

Den siste kontrollen utføres av funksjonen `dropLongURL`. En URL som er klassifisert som privat, lagrer nettleseren permanent i et Bloom-filter og kontrollerer den ikke på nytt.

## Regelen: ingen orddel over 18 tegn

`dropLongURL` deler stien ved skråstrek, punktum, understrek, mellomrom, bindestrek, kolon, pluss og semikolon og sammenligner hver del med grenseverdien `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` er satt til `18` i kildekoden. `return true` betyr: URL-en anses som mistenkelig. Den dekodede URL-en kontrolleres: Nettleseren prosentkoder ikke-ASCII-tegn (`%C3%BC`), WDP konverterer dem først tilbake med `cleanCurrentUrl` (`decodeURIComponent`). Det telles JavaScript-tegn, altså UTF-16-kodeenheter: En `ü` eller et kinesisk skrifttegn teller enkelt, mens et tegn utenfor Unicode-basiskplanet eller en aksent som er lagt til som eget tegn teller dobbelt. Regelen gjelder i begge kontrollmoduser, den vanlige og den strenge. Brave omtaler selv heuristikkene i README som konservative: Mange offentlige sider blir feilaktig klassifisert som mulige hemmelige lenker, noe som aksepteres for WDPs formål.

Bindestreken skiller, et sammenskrevet ord gjør det ikke. Det avgjørende er altså lengden på det lengste enkeltordet i slugen, ikke lengden på hele slugen:

| Slug | Lengste del | Resultat |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | rapporteres |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | forkastes |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | forkastes |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | rapporteres |

## Hvorfor tyske sluger er spesielt berørt

Engelske faguttrykk skrives hver for seg, tyske settes sammen. I tillegg kommer omskrivingen av omlyder: `ü` blir til `ue`, og hvert ord med omlyd blir dermed lengre. `Zertifikatserneuerung` har 21 tegn, `Verschlüsselungsgateway` har 24 i ASCII-omskriving. I skandinaviske språk er det liknende: Svensk og norsk skriver også sammensetninger sammen.

På dette nettstedet er 24 av 750 URL-er berørt etter en analyse av nettstedskartet: tre tyske artikler, én engelsk og 20 svenske eller norske oversettelser. Det engelske tilfellet stammer fra header-feltet `MessageDirectionality`, som ble tatt med som ord i slugen.

## Når canonical hjelper

Hvis den åpnede URL-en feiler på `dropLongURL`, kontrollerer nettleseren sidens canonical-URL. Består den kontrollen, rapporterer den siden under canonical-URL-en. Dette dekker det typiske tilfellet der en side åpnes med sporingsparametere, men har en ren canonical.

Ved en for lang orddel i slugen hjelper dette ikke: Canonical-URL-en er som regel identisk med den åpnede URL-en og feiler på samme regel. Hvis begge feiler, forkaster nettleseren siden.

Om den strenge kontrollmodusen brukes, avhenger også av canonical: Den lempes bare dersom canonical avviker fra den åpnede URL-en og er kortere. For en side som oppgir seg selv som canonical, gjelder streng modus.

## Flere heuristikker i dropLongURL

I tillegg til lengden på orddelene forkaster `dropLongURL` blant annet en URL ved

- en query-streng med mer enn 30 tegn eller mer enn fire parametere (i streng modus fra 23 tegn eller to parametere),
- en siffersekvens med mer enn 12 sifre i stien eller query-strengen (i streng modus mer enn 8); spesialtegn fjernes på forhånd, så en datosti som `/2026/09/25/` teller som et åttesifret tall,
- orddeler som en Markov-klassifikator klassifiserer som hash,
- stisegmenter som `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` eller `/token`,
- en e-postadresse i URL-en.

For en vanlig artikkel-URL uten parametere er som regel bare ordlengden relevant.

## Vurdering: quorum og crawler

Regelen gjelder bare veien via WDP. To punkter begrenser betydningen:

- **Quorum:** Brave kan først dekryptere en siderapport når mer enn et bestemt antall brukere har rapportert samme URL innen 30 dager fra forskjellige nettverk. README oppgir ikke terskelverdien. For sider med få Brave-besøk har WDP derfor uansett liten effekt.
- **Crawler:** Brave-crawleren arbeider uavhengig av WDP. Den opptrer uten egen user-agent og følger robots.txt-reglene for Googlebot. En URL med lang orddel kan likevel komme inn i indeksen via crawleren.

En lang orddel fjerner dermed én av de to veiene inn i Brave-indeksen; siden er fortsatt tilgjengelig via crawleren.

## Kontroller egne URL-er

Følgende kommandoer leser et nettstedskart og skriver ut hver URL der stien inneholder en del med mer enn 18 tegn. Ved en nettstedskartindeks må de enkelte nettstedskartene (`sitemap-0.xml` osv.) kontrolleres etter hverandre.

På Linux eller macOS:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `curl -s` | Laster ned nettstedskartet uten fremdriftsvisning. |
| `grep -oE '<loc>[^<]+'` | Skriver bare ut `<loc>`-oppføringer (`-o`), med utvidede regulære uttrykk (`-E`). |
| `sed -E 's#…##'` | Fjerner `<loc>`, skjema og vertsnavn; bare stien gjenstår. |
| `awk -F'[/._ :+;-]'` | Deler stien ved de samme skilletegnene som `dropLongURL`. |
| `length($i) > 18` | Skriver ut lengde, orddel og sti så snart en del overskrider grenseverdien. |

</details>

På Windows med PowerShell:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Invoke-RestMethod -Uri` | Laster ned nettstedskartet og returnerer det direkte som XML-objekt. |
| `$sitemap.urlset.url.loc` | Leser alle `<loc>`-oppføringer fra XML-en. |
| `([uri]$loc).AbsolutePath` | Fjerner skjema og vertsnavn og leverer stien. |
| `[uri]::UnescapeDataString()` | Dekoder prosentkodingen, slik WDP også gjør før kontrollen. |
| `-split '[/._ :+;-]'` | Deler stien ved de samme skilletegnene som `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Beholder bare orddeler med mer enn 18 tegn. |

</details>

Begge variantene kontrollerer bare ordlengden, ikke de øvrige heuristikkene. Bash-varianten teller den prosentkodede formen og gir derfor bare korrekte verdier for rene ASCII-sluger; bruk PowerShell-varianten for sluger med omlyder eller andre skriftsystemer.

## Tilpass sluger

For nye artikler kan regelen overholdes når slugen fastsettes: Skill sammensetninger i slugen med bindestrek (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) og forkort betegnelser hentet fra faguttrykk.

For eksisterende URL-er må en endring vurderes. Hver slug-endring trenger en 301-videresending fra den gamle til den nye URL-en, et oppdatert nettstedskart og tilpassede interne lenker. For sider som allerede rangerer godt eller har eksterne lenker, gir omleggingen lite: De er uansett i indeksen via crawleren. Den er først og fremst fornuftig for nye sider som ennå ikke finnes i noen indeks.

## Måling: Hvor sterkt er enkelte språk berørt?

Om regelen rammer språk ulikt, kan måles ved å sende de samme begrepene på forskjellige språk gjennom Braves originalkode. Skript, data og enkeltresultater finnes i det offentlige repositoriet [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); målingen kan reproduseres der med fem kommandoer.

**Oppsett:**

- **Kode:** Braves repositorium `web-discovery-project`, commit `58b1b53` fra 9. september 2026. Funksjonene `cleanCurrentUrl` og `dropLongURL` kjører uendret i Node.js; bare koblingene til nettleser og lagring er erstattet. Braves egne testtilfeller for `dropLongURL` gir dermed de forventede resultatene.
- **Begreper:** Listen «Vital Articles Level 3» fra engelske Wikipedia, rundt 1000 sentrale artikler. Via Wikipedia-språklenkene ble titlene på de samme artiklene hentet på 23 andre språk. 825 begreper finnes på alle 24 språkene. For hvert begrep er dermed bare språket endret.
- **URL-er:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, dannet som i nettleseren (prosentkodet), deretter som i WDP først gjennom `cleanCurrentUrl` og så gjennom `dropLongURL` i vanlig og streng modus.
- **Årsak:** Hver forkastede URL ble kontrollert en gang til med lengderegelen deaktivert. Dersom den da rapporteres, skyldes forkastelsen lengderegelen.
- **Statistikk:** 95 %-konfidensintervall etter Wilson; sammenligning med engelsk gjennom de parede begrepene (eksakt McNemar-test).

Kjernen i analysen:

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

**Resultat** (vanlig modus, 825 begreper per språk):

| Språk | Forkastet | 95 %-KI | herav lengderegelen | p mot engelsk |
|---|---|---|---|---|
| Engelsk | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Tysk | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Ungarsk | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Nederlandsk | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finsk | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Svensk | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Norsk | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Dansk, fransk, spansk, portugisisk, tyrkisk, polsk | hver 1 (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Russisk, ukrainsk, japansk | hver 1 (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italiensk, gresk, arabisk, hebraisk, persisk, hindi, kinesisk, koreansk | 0 (0,0 %) | 0,0–0,5 % | – | – |

Blant eksemplene på forkastede URL-er er `Schwangerschaftsabbruch`, `Empfängnisverhütung` og `Ingenieurwissenschaften` (tysk), `Terhességmegszakítás` (ungarsk), `Milieuverontreiniging` (nederlandsk) og `Tietojenkäsittelytiede` (finsk). Enkelttilfellene i de øvrige språkene gjelder nesten alle samme begrep: deoksyribonukleinsyre på det aktuelle landsspråket. I ingen tilfeller ble et begrep på landsspråket rapportert og forkastet på engelsk.

**Analyse:**

- Språk med ikke-latinsk skrift diskrimineres ikke. Brave dekoder URL-en før kontrollen, slik at et kyrillisk eller kinesisk tegn teller som ett tegn.
- Språk som skriver sammensetninger sammen, har en liten, men systematisk ulempe. For tysk er den med p = 0,002 også signifikant når man tar hensyn til at 23 språk sammenlignes samtidig (Bonferroni-terskel 0,0022). Ungarsk ligger så vidt over.
- Den målte andelen gjelder Wikipedia-titler, som som regel består av ett eller to ord. Blogg-sluger inneholder flere faguttrykk; på dette nettstedet er 3 av 66 tyske artikkel-URL-er berørt (4,5 %).

**Begrensninger:** Det var kontrollregelen som ble målt, ikke faktisk opptak i Brave-indeksen. Wikipedia selv står trolig på Braves tillatelsesliste og er i praksis ikke berørt; målingen viser hvordan regelen behandler et nettsted uten tillatelse med slike URL-er. Den offentlige commiten behøver ikke samsvare med versjonen som leveres i nettleseren. Målingen gjengir ikke omveien via canonical-URL-en.

## Mening: En lengdegrense som diskriminerer språk med sammensetninger

*Dette avsnittet gjengir forfatterens vurdering. Faktaene bak står i avsnittene over.*

Grensen på 18 tegn teller tegn og rammer dermed språk som skriver begreper sammen. Målingen viser effekten: For identiske begreper forkaster regelen 1,2 % av de tyske og 0 % av de engelske URL-ene, og hver av disse tyske forkastelsene skyldes lengderegelen. Den engelske slugen `data-protection-regulation` består kontrollen, mens `datenschutzgrundverordnung` ikke gjør det. Effekten er liten, men rammer alltid de samme språkene: tysk, ungarsk, nederlandsk, finsk og de skandinaviske språkene. Etter mitt syn er dette en diskriminering av disse språkene, selv om den ikke er tilsiktet.

### Hvem som bærer kostnadene

Brave skriver i README at feilklassifiseringer ikke er et stort problem for eget formål. For Brave stemmer det: En forkastet side koster Brave én rapport. For de berørte nettstedene er det en systematisk ulempe som særlig rammer fagtekster, fordi nettopp faguttrykk danner lange sammensetninger. Fordi Claudes nettsøk bygger på Brave-indeksen, mangler disse sidene én vei inn i indeksen Claude siterer fra.

I tillegg finnes det en tillatelsesliste på serversiden: URL-mønstre som Brave godtar der (`allowlisted`), hopper over kontrollen. Hvilke mønstre dette er, er ikke offentlig dokumentert. Enkeltstående fagnettsteder kan ikke påvirke dette.

### Hva som taler for regelen

Formålet er legitimt. Delingslenker med token, for eksempel for delte dokumenter, er en reell risiko, og en slik lenke i søkeindeksen ville være en alvorlig personvernkrenkelse. En hard lengdegrense er enkel, rask og vanskelig å omgå. Den kan heller ikke uten videre erstattes av hash-gjenkjenning: I en tilleggstest med 2000 tilfeldige tokener med små bokstaver og 22 tegn, klassifiserte Braves `isHash` bare 58 % som hash, og 94 % for tokener med små bokstaver og sifre. Alle testede sammensetninger som `schwangerschaftsabbruch` eller `datenschutzgrundverordnung` ble korrekt gjenkjent av funksjonen som ikke-hash. For hemmelige lenker med bare små bokstaver er lengdegrensen dermed den egentlige beskyttelsen. Crawleren når dessuten fortsatt berørte sider, og den målte andelen er liten.

### Hva Brave kunne endre

- **Skille ord fra tokener:** En enkel heving av grensen for ord med bare bokstaver ville, ifølge tilleggstesten, slippe gjennom hemmelige lenker med små bokstaver. En ekstra plausibilitetskontroll bare for orddeler mellom 19 og omtrent 30 tegn kan være tenkelig, for eksempel basert på rekkefølgen av vokaler og konsonanter eller en liten språkmodell trent på ord i de berørte språkene. Om dette er pålitelig nok, må Brave undersøke.
- **Dokumentere regelen:** Hjelpesiden om crawleren nevner WDP, men ikke heuristikkene. Et hint til nettstedsoperatører ville være nok til at de kan tilpasse slugene sine etter dette.

Jeg har meldt målingen og denne målkonflikten til Brave som [Issue #498](https://github.com/brave/web-discovery-project/issues/498) i WDP-repositoriet. Inntil da gjenstår bare tilpasning på nettstedets side: Skill sammensetninger i slugen med bindestrek. At dette arbeidet ligger hos operatører i bestemte språkområder og ikke hos metoden, er kjernen i min kritikk.

*Korrigering fra 25.09.2026: En tidligere versjon foreslo å heve grensen generelt for ord med bare bokstaver; tilleggstesten for hash-gjenkjenning viser at dette ville svekke beskyttelsen. En tidligere versjon av dette avsnittet hevdet at ikke-latinske skriftsystemer og diakritiske tegn ville diskrimineres spesielt sterkt gjennom prosentkoding, basert på en måling med prosentkodede URL-er. En uavhengig etterprøving viste at Brave dekoder URL-er med `cleanCurrentUrl` før kontrollen. Påstanden var feil og er fjernet; målingen over bruker korrekt forløp.*

## Kilder

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): Beskrivelse av WDP fra Brave, med meldingstyper, ny henting uten informasjonskapsler, Capability-URL-heuristikker og quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): Kildekode med `dropLongURL`, `calculateStrictness` og grenseverdiene `rel_part_len: 18` og `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): Offisiell hjelpeside om crawleren, den manglende egne user-agenten og WDPs rolle.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): Oppføring av Brave Search i Anthropics liste over underleverandører i mars 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): Rapport med ytterligere indikasjoner, blant annet parameteren `BraveSearchParams` i Claudes nettsøk.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): Dekoding av URL-en før kontrollen; kommentaren i onLocationChange (linje 1724) beskriver den dekodede URL-en som WDPs interne representasjon.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): Begrepslisten for målingen; titlene på de øvrige språkene kommer fra språklenkene i Wikipedia-API-et.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): Måleskript, datasett og enkeltresultater for språkmålingen i denne artikkelen, inkludert tilleggstesten for hash-gjenkjenning (`hash-test.mjs`) og den første versjonen som er merket som feilaktig.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): Tilbakemelding til Brave med måleresultatene og målkonflikten mellom lengdegrense og hash-gjenkjenning.
