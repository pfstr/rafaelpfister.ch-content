---
title: "Brave Web Discovery Project: perché le parti di parole lunghe negli URL escludono le pagine"
navTitle: "Brave: parti URL lunghe"
description: "Nel Web Discovery Project, Brave scarta ogni URL il cui percorso contiene una parte di parola con più di 18 caratteri. Composti tedeschi come Verschlüsselungsgateway rientrano in questa categoria. Poiché la ricerca web di Claude si basa sull’indice di Brave, ciò riguarda anche la visibilità in Claude. La regola nel codice sorgente, altre euristiche, il ruolo del canonical, uno script di verifica per la propria sitemap, una misurazione con 825 termini in 24 lingue e una valutazione del motivo per cui la regola svantaggia le lingue con composti."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min di lettura"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-perche-le-parti-di-parole-lunghe-negli-url-escludono-le-pagine"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/it/blog/brave-web-discovery-project-perche-le-parti-di-parole-lunghe-negli-url-escludono-le-pagine
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T08:58:14.908Z
translationReview: automatic
---

# Brave Web Discovery Project: perché le parti di parole lunghe negli URL escludono le pagine

La ricerca web di Claude fornisce i suoi risultati dall’indice di Brave Search. Anthropic indica Brave Search come subfornitore dal marzo 2025 e le citazioni nelle risposte di Claude corrispondono in larga misura ai risultati di Brave. Chi vuole essere citato nelle risposte di Claude deve quindi figurare nell’indice di Brave. Le segnalazioni tramite Google Search Console, Bing Webmaster Tools o IndexNow non raggiungono Brave direttamente.

Brave alimenta il proprio indice da due fonti: un crawler proprio e il Web Discovery Project (WDP). Il WDP è una funzione opt-in del browser Brave che segnala anonimamente a Brave le pagine visitate. Prima che il browser segnali una pagina, verifica l’URL mediante una serie di euristiche. Una di esse riguarda in particolare i siti web in lingua tedesca: se il percorso contiene una parte di parola con più di 18 caratteri, la pagina non viene mai segnalata.

## Cosa segnala il Web Discovery Project

Il WDP raccoglie due tipi di dati: query di ricerca su Google, Bing, Brave e alcuni altri motori di ricerca, compreso l’elenco dei risultati, e visualizzazioni di pagine con URL, titolo, tempo di permanenza e interazione. Non esistono ID utente; ogni segnalazione viene inviata singolarmente a Brave, in forma crittografata e distribuita casualmente nel tempo.

Affinché non vengano segnalati contenuti privati, il browser classifica una pagina come privata se

- una seconda richiesta senza cookie restituisce una pagina sensibilmente diversa (login, personalizzazione),
- il dominio punta a un indirizzo IP privato,
- la pagina contiene `noindex`,
- l’URL assomiglia a un link segreto (URL capability, ad esempio un link di condivisione con token).

L’ultimo controllo è eseguito dalla funzione `dropLongURL`. Un URL classificato come privato viene memorizzato in modo permanente dal browser in un filtro Bloom e non viene controllato di nuovo.

## La regola: nessuna parte di parola oltre 18 caratteri

`dropLongURL` divide il percorso a barre, punti, trattini bassi, spazi, trattini, due punti, segni più e punti e virgola e confronta ogni parte con il valore limite `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` è impostato nel codice sorgente su `18`. `return true` significa: l’URL è considerato sospetto. Viene controllato l’URL decodificato: il browser codifica in percentuale i caratteri non ASCII (`%C3%BC`), il WDP li riconverte prima con `cleanCurrentUrl` (`decodeURIComponent`). Si contano i caratteri JavaScript, cioè le unità di codice UTF-16: una `ü` o un carattere cinese conta una sola volta, un carattere esterno al piano Unicode di base o un accento aggiunto come carattere separato conta due volte. La regola si applica in entrambe le modalità di controllo, quella normale e quella rigorosa. Nel README Brave definisce le euristiche stesse conservative: molte pagine pubbliche vengono erroneamente classificate come possibili link segreti, cosa che viene accettata per lo scopo del WDP.

Il trattino separa, una parola scritta tutta unita no. È quindi determinante la lunghezza della parola singola più lunga nello slug, non la lunghezza dell’intero slug:

| Slug | Parte più lunga | Risultato |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | viene segnalato |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | scartato |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | scartato |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | viene segnalato |

## Perché gli slug tedeschi sono particolarmente colpiti

I termini tecnici inglesi si scrivono separati, quelli tedeschi composti. A ciò si aggiunge la traslitterazione delle dieresi: da `ü` diventa `ue`, rendendo più lunga ogni parola con dieresi. `Zertifikatserneuerung` ha 21 caratteri, `Verschlüsselungsgateway` nella traslitterazione ASCII 24. Nelle lingue scandinave la situazione è simile: anche svedese e norvegese uniscono i composti.

Su questo sito web, secondo un’analisi della sitemap, sono interessati 24 URL su 750: tre articoli tedeschi, uno inglese e 20 traduzioni svedesi o norvegesi. Il caso inglese deriva dal campo header `MessageDirectionality`, adottato come parola nello slug.

## Quando il canonical aiuta

Se l’URL richiamato non supera `dropLongURL`, il browser verifica l’URL canonical della pagina. Se quest’ultimo supera il controllo, segnala la pagina con l’URL canonical. Ciò copre il caso tipico in cui una pagina viene richiamata con parametri di tracciamento, ma possiede un canonical pulito.

Con una parte di parola troppo lunga nello slug, questo non aiuta: l’URL canonical è generalmente identico all’URL richiamato e non supera la stessa regola. Se entrambi non superano il controllo, il browser scarta la pagina.

Anche l’applicazione della modalità di controllo rigorosa dipende dal canonical: viene allentata solo se il canonical differisce dall’URL richiamato ed è più corto. Per una pagina che indica sé stessa come canonical, vale la modalità rigorosa.

## Altre euristiche in dropLongURL

Oltre alla lunghezza delle parti di parola, `dropLongURL` scarta un URL, tra l’altro, in caso di

- una query string con più di 30 caratteri o più di quattro parametri (in modalità rigorosa da 23 caratteri o due parametri),
- una sequenza di cifre con più di 12 cifre nel percorso o nella query string (in modalità rigorosa più di 8); prima vengono rimossi i caratteri speciali, quindi un percorso data come `/2026/09/25/` conta come un numero di otto cifre,
- parti di parola che un classificatore Markov classifica come hash,
- segmenti di percorso come `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` o `/token`,
- un indirizzo e-mail nell’URL.

Per un normale URL di articolo senza parametri, di solito è rilevante soltanto la lunghezza delle parole.

## Inquadramento: quorum e crawler

La regola riguarda soltanto il percorso tramite WDP. Due aspetti ne limitano l’importanza:

- **Quorum:** Brave può decifrare una segnalazione di pagina solo quando più di un determinato numero di utenti ha segnalato lo stesso URL entro 30 giorni da reti diverse. Il README non indica la soglia. Per le pagine con pochi visitatori Brave, il WDP ha quindi comunque poca rilevanza.
- **Crawler:** il crawler di Brave opera indipendentemente dal WDP. Non si presenta con un proprio user agent e segue le regole robots.txt per Googlebot. Un URL con una parte di parola lunga può comunque entrare nell’indice tramite il crawler.

Una parte di parola lunga priva quindi una pagina di una delle due vie verso l’indice di Brave; resta raggiungibile tramite il crawler.

## Verificare i propri URL

I seguenti comandi leggono una sitemap e restituiscono ogni URL il cui percorso contiene una parte con più di 18 caratteri. In caso di indice sitemap, verificare una dopo l’altra le singole sitemap (`sitemap-0.xml` ecc.).

Su Linux o macOS:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `curl -s` | Scarica la sitemap senza indicatore di avanzamento. |
| `grep -oE '<loc>[^<]+'` | Restituisce solo le voci `<loc>` (`-o`), con espressioni regolari estese (`-E`). |
| `sed -E 's#…##'` | Rimuove `<loc>`, schema e nome host; resta il percorso. |
| `awk -F'[/._ :+;-]'` | Divide il percorso con gli stessi separatori di `dropLongURL`. |
| `length($i) > 18` | Restituisce lunghezza, parte di parola e percorso non appena una parte supera il valore limite. |

</details>

Su Windows con PowerShell:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Invoke-RestMethod -Uri` | Scarica la sitemap e la restituisce direttamente come oggetto XML. |
| `$sitemap.urlset.url.loc` | Legge tutte le voci `<loc>` dall’XML. |
| `([uri]$loc).AbsolutePath` | Rimuove schema e nome host e restituisce il percorso. |
| `[uri]::UnescapeDataString()` | Decodifica la codifica percentuale, come fa anche il WDP prima del controllo. |
| `-split '[/._ :+;-]'` | Divide il percorso con gli stessi separatori di `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Mantiene solo le parti di parola con più di 18 caratteri. |

</details>

Entrambe le varianti verificano solo la lunghezza delle parole, non le altre euristiche. La variante Bash conta la forma codificata in percentuale e pertanto fornisce valori corretti solo per slug puramente ASCII; per slug con dieresi o altri sistemi di scrittura, usare la variante PowerShell.

## Adattare gli slug

Per i nuovi articoli, la regola può essere rispettata quando si definisce lo slug: separare con trattini i composti nello slug (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) e abbreviare le denominazioni riprese da termini tecnici.

Per gli URL esistenti, una modifica va ponderata. Ogni modifica dello slug richiede un reindirizzamento 301 dal vecchio al nuovo URL, una sitemap aggiornata e link interni adattati. Per le pagine che già si posizionano bene o ricevono link esterni, il cambiamento apporta poco: sono comunque nell’indice tramite il crawler. È sensato soprattutto per pagine recenti che non figurano ancora in alcun indice.

## Misurazione: quanto sono colpite le singole lingue?

Se la regola colpisca le lingue in modo diverso può essere misurato facendo passare gli stessi termini in varie lingue attraverso il codice originale di Brave. Script, dati e risultati individuali sono disponibili nel repository pubblico [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); la misurazione può essere riprodotta lì con cinque comandi.

**Impostazione:**

- **Codice:** repository di Brave `web-discovery-project`, commit `58b1b53` del 9 settembre 2026. Le funzioni `cleanCurrentUrl` e `dropLongURL` vengono eseguite invariate in Node.js; sono stati sostituiti solo i collegamenti al browser e alla memoria. I test di Brave per `dropLongURL` forniscono quindi i risultati attesi.
- **Termini:** l’elenco “Vital Articles Level 3” della Wikipedia inglese, circa 1000 articoli centrali. Attraverso i collegamenti linguistici di Wikipedia sono stati recuperati i titoli degli stessi articoli in altre 23 lingue. Esistono 825 termini in tutte le 24 lingue. Per ogni termine cambia quindi solo la lingua.
- **URL:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, creati come nel browser (codificati in percentuale), poi passati come nel WDP prima attraverso `cleanCurrentUrl` e successivamente attraverso `dropLongURL` in modalità normale e rigorosa.
- **Causa:** ogni URL scartato è stato verificato una seconda volta, con la regola di lunghezza disattivata. Se veniva allora segnalato, lo scarto era dovuto alla regola di lunghezza.
- **Statistica:** intervallo di confidenza al 95% secondo Wilson; confronto con l’inglese tramite termini appaiati (test esatto di McNemar).

Il nucleo dell’analisi:

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

**Risultato** (modalità normale, 825 termini per lingua):

| Lingua | Scartati | IC 95% | di cui regola di lunghezza | p rispetto all’inglese |
|---|---|---|---|---|
| Inglese | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Tedesco | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Ungherese | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Olandese | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finlandese | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Svedese | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Norvegese | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Danese, francese, spagnolo, portoghese, turco, polacco | 1 ciascuno (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Russo, ucraino, giapponese | 1 ciascuno (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italiano, greco, arabo, ebraico, persiano, hindi, cinese, coreano | 0 (0,0 %) | 0,0–0,5 % | – | – |

Sono stati scartati, per esempio, `Schwangerschaftsabbruch`, `Empfängnisverhütung` e `Ingenieurwissenschaften` (tedesco), `Terhességmegszakítás` (ungherese), `Milieuverontreiniging` (olandese) e `Tietojenkäsittelytiede` (finlandese). I singoli casi nelle altre lingue riguardano quasi tutti lo stesso termine: acido desossiribonucleico nella rispettiva lingua nazionale. In nessun caso un termine è stato segnalato nella lingua nazionale e scartato in inglese.

**Analisi:**

- Le lingue con scrittura non latina non sono svantaggiate. Brave decodifica l’URL prima del controllo, per cui un carattere cirillico o cinese conta come un carattere.
- Le lingue che scrivono i composti uniti hanno uno svantaggio piccolo ma sistematico. Per il tedesco è significativo con p = 0,002 anche considerando che vengono confrontate simultaneamente 23 lingue (soglia Bonferroni 0,0022). L’ungherese si colloca appena al di sopra.
- La quota misurata vale per titoli Wikipedia, che solitamente sono composti da una o due parole. Gli slug dei blog contengono più termini tecnici; su questo sito sono interessati 3 dei 66 URL di articoli tedeschi (4,5 %).

**Limiti:** è stata misurata la regola di controllo, non l’effettiva inclusione nell’indice di Brave. Wikipedia stessa è presumibilmente nella lista di autorizzazione di Brave e nella pratica non è interessata; la misurazione mostra come la regola tratta un sito web senza autorizzazione con tali URL. Il commit pubblico potrebbe non corrispondere alla versione distribuita nel browser. La misurazione non riproduce il percorso alternativo tramite l’URL canonical.

## Opinione: un limite di lunghezza che svantaggia le lingue con composti

*Questa sezione riporta la valutazione dell’autore. I fatti in merito sono riportati nelle sezioni precedenti.*

Il limite di 18 caratteri conta i caratteri e colpisce quindi le lingue che uniscono i termini. La misurazione mostra l’effetto: a parità di termini, la regola scarta l’1,2 % degli URL tedeschi e lo 0 % di quelli inglesi, e ciascuno di questi scarti tedeschi è dovuto alla regola di lunghezza. Lo slug inglese `data-protection-regulation` supera il controllo, `datenschutzgrundverordnung` no. L’effetto è piccolo, ma colpisce sempre le stesse lingue: tedesco, ungherese, olandese, finlandese e lingue scandinave. A mio avviso, si tratta di uno svantaggio per queste lingue, anche se non intenzionale.

### Chi sostiene i costi

Brave scrive nel README che le classificazioni errate non sono un grande problema per il proprio scopo. Per Brave è vero: una pagina scartata costa a Brave una segnalazione. Per i siti web interessati è uno svantaggio sistematico, che colpisce soprattutto i testi tecnici perché proprio i termini tecnici formano composti lunghi. Poiché la ricerca web di Claude si basa sull’indice di Brave, per queste pagine viene meno una via verso l’indice da cui Claude cita.

Si aggiunge una lista di autorizzazione lato server: i modelli URL che Brave vi inserisce (`allowlisted`) saltano il controllo. Non è documentato pubblicamente quali siano questi modelli. I singoli siti tecnici non possono influenzarli.

### Cosa parla a favore della regola

Lo scopo è legittimo. I link di condivisione con token, ad esempio per documenti condivisi, sono un rischio reale e un simile link nell’indice di ricerca costituirebbe un grave incidente di protezione dei dati. Un limite di lunghezza rigido è semplice, rapido e difficile da aggirare. Non può neppure essere semplicemente sostituito dal rilevamento degli hash: in un test aggiuntivo con 2000 token casuali di 22 caratteri composti da lettere minuscole, `isHash` di Brave ha classificato solo il 58 % come hash, mentre per token composti da lettere minuscole e cifre la percentuale era del 94 %. Tutti i composti testati come `schwangerschaftsabbruch` o `datenschutzgrundverordnung` sono stati correttamente riconosciuti dalla funzione come non hash. Per i link segreti composti solo da lettere minuscole, il limite di lunghezza è quindi la protezione effettiva. Inoltre, il crawler continua a raggiungere le pagine interessate e la quota misurata è piccola.

### Cosa potrebbe cambiare Brave

- **Distinguere le parole dai token:** un semplice aumento del limite per le parole composte solo da lettere lascerebbe passare, secondo il test aggiuntivo, i link segreti composti da lettere minuscole. Si potrebbe prevedere un controllo aggiuntivo di plausibilità solo per le parti di parola tra 19 e circa 30 caratteri, ad esempio mediante la sequenza di vocali e consonanti o un piccolo modello linguistico addestrato su parole delle lingue interessate. Brave dovrebbe verificare se ciò sia sufficientemente affidabile.
- **Documentare la regola:** la pagina di aiuto sul crawler menziona il WDP, ma non le euristiche. Un avviso per i gestori di siti web basterebbe affinché possano adattare i propri slug di conseguenza.

Ho segnalato a Brave questa misurazione e questo conflitto di obiettivi come [Issue #498](https://github.com/brave/web-discovery-project/issues/498) nel repository del WDP. Fino ad allora resta solo l’adattamento dal lato del sito web: separare con trattini i composti nello slug. Il fatto che questo lavoro ricada sui gestori di determinate aree linguistiche e non sul metodo è il fulcro della mia critica.

*Correzione del 25.09.2026: una versione precedente proponeva di aumentare indiscriminatamente il limite per le parole composte solo da lettere; il test aggiuntivo sul rilevamento degli hash mostra che ciò indebolirebbe la protezione. Una versione precedente di questa sezione affermava che le scritture non latine e i segni diacritici sarebbero particolarmente svantaggiati dalla codifica percentuale, sulla base di una misurazione con URL codificati in percentuale. Una verifica indipendente ha mostrato che Brave decodifica gli URL prima del controllo con `cleanCurrentUrl`. L’affermazione era errata ed è stata rimossa; la misurazione sopra utilizza il flusso corretto.*

## Fonti

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): descrizione del WDP da parte di Brave, con tipi di messaggi, seconda richiesta senza cookie, euristiche per URL capability e quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): codice sorgente con `dropLongURL`, `calculateStrictness` e i valori limite `rel_part_len: 18` e `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): pagina di aiuto ufficiale sul crawler, sull’assenza di un proprio user agent e sul ruolo del WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): inserimento di Brave Search nell’elenco dei subfornitori di Anthropic nel marzo 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): articolo con ulteriori indizi, come il parametro `BraveSearchParams` nella ricerca web di Claude.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): decodifica dell’URL prima del controllo; il commento in onLocationChange (riga 1724) descrive l’URL decodificato come rappresentazione interna del WDP.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): elenco di termini della misurazione; i titoli nelle altre lingue provengono dai collegamenti linguistici dell’API di Wikipedia.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): script di misurazione, set di dati e risultati individuali della misurazione linguistica in questo articolo, incluso il test aggiuntivo sul rilevamento degli hash (`hash-test.mjs`) e la prima versione contrassegnata come errata.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): segnalazione a Brave con i risultati della misurazione e il conflitto di obiettivi tra limite di lunghezza e rilevamento degli hash.
