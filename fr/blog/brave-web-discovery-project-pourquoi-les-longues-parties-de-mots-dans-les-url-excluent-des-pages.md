---
title: "Brave Web Discovery Project : pourquoi les longues parties de mots dans les URL excluent des pages"
navTitle: "Brave : longues parties d’URL"
description: "Dans le Web Discovery Project, Brave rejette toute URL dont le chemin contient une partie de mot de plus de 18 caractères. Les composés allemands tels que Verschlüsselungsgateway sont concernés. Comme la recherche web de Claude repose sur l’index de Brave, cela affecte également la visibilité dans Claude. La règle dans le code source, d’autres heuristiques, le rôle de l’URL canonique, un script de vérification pour son propre sitemap, une mesure portant sur 825 termes dans 24 langues et une évaluation expliquant pourquoi la règle désavantage les langues à mots composés."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min de lecture"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-pourquoi-les-longues-parties-de-mots-dans-les-url-excluent-des-pages"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/fr/blog/brave-web-discovery-project-pourquoi-les-longues-parties-de-mots-dans-les-url-excluent-des-pages
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T08:57:12.908Z
translationReview: required
---

# Brave Web Discovery Project : pourquoi les longues parties de mots dans les URL excluent des pages

La recherche web de Claude fournit ses résultats à partir de l’index de Brave Search. Anthropic cite Brave Search comme sous-traitant depuis mars 2025, et les citations dans les réponses de Claude correspondent largement aux résultats de Brave. Quiconque souhaite être cité dans les réponses de Claude doit donc figurer dans l’index de Brave. Les soumissions via Google Search Console, Bing Webmaster Tools ou IndexNow n’atteignent pas Brave directement.

Brave alimente son index à partir de deux sources : son propre crawler et le Web Discovery Project (WDP). Le WDP est une fonction facultative du navigateur Brave qui signale anonymement à Brave les pages visitées. Avant que le navigateur ne signale une page, il vérifie l’URL à l’aide d’une série d’heuristiques. L’une d’elles concerne particulièrement les sites web germanophones : si le chemin contient une partie de mot de plus de 18 caractères, la page n’est jamais signalée.

## Ce que signale le Web Discovery Project

Le WDP collecte deux types de données : les requêtes de recherche sur Google, Bing, Brave et quelques autres moteurs de recherche avec leur liste de résultats, ainsi que les visites de pages avec l’URL, le titre, le temps passé et les interactions. Il n’y a pas d’identifiants utilisateur ; chaque signalement est envoyé individuellement, chiffré et réparti aléatoirement dans le temps à Brave.

Afin qu’aucun contenu privé ne soit signalé, le navigateur considère une page comme privée lorsque

- une seconde consultation sans cookies fournit une page nettement différente (connexion, personnalisation),
- le domaine pointe vers une adresse IP privée,
- la page porte `noindex`,
- l’URL ressemble à un lien secret (URL à capacité, par exemple un lien de partage avec jeton).

La dernière vérification est assurée par la fonction `dropLongURL`. Une URL considérée comme privée est enregistrée durablement par le navigateur dans un filtre de Bloom et n’est plus vérifiée.

## La règle : aucune partie de mot de plus de 18 caractères

`dropLongURL` découpe le chemin aux barres obliques, points, traits de soulignement, espaces, traits d’union, deux-points, signes plus et points-virgules, puis compare chaque partie à la limite `rel_part_len` :

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` est fixé dans le code source à `18`. `return true` signifie que l’URL est considérée comme suspecte. C’est l’URL décodée qui est vérifiée : le navigateur encode les caractères non ASCII en pourcentage (`%C3%BC`), le WDP les reconvertit auparavant avec `cleanCurrentUrl` (`decodeURIComponent`). Sont comptés les caractères JavaScript, donc les unités de code UTF-16 : un `ü` ou un caractère chinois compte simplement, tandis qu’un caractère hors du plan multilingue de base Unicode ou un accent ajouté comme caractère distinct compte double. La règle s’applique dans les deux modes de vérification, normal et strict. Dans le README, Brave qualifie lui-même ces heuristiques de conservatrices : de nombreuses pages publiques sont à tort classées comme de possibles liens secrets, ce qui est accepté pour l’objectif du WDP.

Le trait d’union sépare, pas un mot écrit en un seul bloc. Ce qui compte est donc la longueur du mot individuel le plus long dans le slug, et non celle de l’ensemble du slug :

| Slug | Partie la plus longue | Résultat |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | est signalé |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | rejeté |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | rejeté |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | est signalé |

## Pourquoi les slugs allemands sont particulièrement touchés

Les termes techniques anglais s’écrivent séparément, les allemands sont composés. À cela s’ajoute la transcription des umlauts : `ü` devient `ue`, ce qui allonge chaque mot contenant un umlaut. `Zertifikatserneuerung` compte 21 caractères, `Verschlüsselungsgateway` 24 en transcription ASCII. La situation est similaire dans les langues scandinaves : le suédois et le norvégien écrivent également les mots composés en un seul bloc.

Sur ce site, une analyse du sitemap montre que 24 URL sur 750 sont concernées : trois articles allemands, un anglais et 20 traductions suédoises ou norvégiennes. Le cas anglais provient du champ d’en-tête `MessageDirectionality`, repris comme mot dans le slug.

## Quand l’URL canonique aide

Si l’URL consultée échoue à `dropLongURL`, le navigateur vérifie l’URL canonique de la page. Si celle-ci passe la vérification, il signale la page sous l’URL canonique. Cela couvre le cas typique d’une page consultée avec des paramètres de suivi, mais ayant une URL canonique propre.

Pour une partie de mot trop longue dans le slug, cela n’aide pas : l’URL canonique est généralement identique à l’URL consultée et échoue à la même règle. Si les deux échouent, le navigateur rejette la page.

L’application du mode de vérification strict dépend également de l’URL canonique : il n’est assoupli que lorsque l’URL canonique diffère de l’URL consultée et est plus courte. Pour une page qui se désigne elle-même comme canonique, le mode strict s’applique.

## Autres heuristiques dans dropLongURL

Outre la longueur des parties de mots, `dropLongURL` rejette notamment une URL en présence

- d’une chaîne de requête de plus de 30 caractères ou de plus de quatre paramètres (en mode strict, dès 23 caractères ou deux paramètres),
- d’une suite de plus de 12 chiffres dans le chemin ou la chaîne de requête (plus de 8 en mode strict) ; les caractères spéciaux sont supprimés auparavant, de sorte qu’un chemin de date comme `/2026/09/25/` compte comme un nombre à huit chiffres,
- de parties de mots qu’un classificateur de Markov considère comme un hash,
- de segments de chemin tels que `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` ou `/token`,
- d’une adresse e-mail dans l’URL.

Pour une URL d’article ordinaire sans paramètres, seule la longueur des mots est généralement pertinente.

## Mise en contexte : quorum et crawler

La règle ne concerne que le chemin via le WDP. Deux points limitent son importance :

- **Quorum :** Brave ne peut déchiffrer un signalement de page que lorsqu’un nombre supérieur à un certain seuil d’utilisateurs a signalé la même URL dans les 30 jours depuis des réseaux différents. Le README ne donne pas ce seuil. Pour les pages ayant peu de visiteurs utilisant Brave, le WDP intervient donc de toute façon peu.
- **Crawler :** Le crawler de Brave fonctionne indépendamment du WDP. Il se présente sans user-agent propre et suit les règles robots.txt applicables à Googlebot. Une URL comportant une longue partie de mot peut malgré tout entrer dans l’index via le crawler.

Une longue partie de mot retire ainsi à une page l’une des deux voies vers l’index de Brave ; elle reste accessible via le crawler.

## Vérifier ses propres URL

Les commandes suivantes lisent un sitemap et affichent chaque URL dont le chemin contient une partie de plus de 18 caractères. Pour un index de sitemaps, vérifier les sitemaps individuels (`sitemap-0.xml` etc.) l’un après l’autre.

Sous Linux ou macOS :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `curl -s` | Télécharge le sitemap sans afficher de progression. |
| `grep -oE '<loc>[^<]+'` | N’affiche que les entrées `<loc>` (`-o`), avec des expressions régulières étendues (`-E`). |
| `sed -E 's#…##'` | Supprime `<loc>`, le schéma et le nom d’hôte ; il ne reste que le chemin. |
| `awk -F'[/._ :+;-]'` | Découpe le chemin aux mêmes séparateurs que `dropLongURL`. |
| `length($i) > 18` | Affiche la longueur, la partie de mot et le chemin dès qu’une partie dépasse la limite. |

</details>

Sous Windows avec PowerShell :

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
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `Invoke-RestMethod -Uri` | Télécharge le sitemap et le renvoie directement sous forme d’objet XML. |
| `$sitemap.urlset.url.loc` | Lit toutes les entrées `<loc>` du XML. |
| `([uri]$loc).AbsolutePath` | Retire le schéma et le nom d’hôte, puis renvoie le chemin. |
| `[uri]::UnescapeDataString()` | Décode l’encodage en pourcentage, comme le fait le WDP avant la vérification. |
| `-split '[/._ :+;-]'` | Découpe le chemin aux mêmes séparateurs que `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Ne conserve que les parties de mots de plus de 18 caractères. |

</details>

Les deux variantes ne vérifient que la longueur des mots, pas les autres heuristiques. La variante Bash compte la forme encodée en pourcentage et ne fournit donc des valeurs correctes que pour les slugs entièrement ASCII ; pour les slugs avec umlauts ou autres écritures, utiliser la variante PowerShell.

## Adapter les slugs

Pour les nouveaux articles, la règle peut être respectée lors de la définition du slug : séparer les composés dans le slug par des traits d’union (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) et raccourcir les désignations reprises de termes techniques.

Pour les URL existantes, une modification doit être pesée. Chaque changement de slug nécessite une redirection 301 de l’ancienne URL vers la nouvelle, un sitemap mis à jour et des liens internes adaptés. Pour les pages déjà bien classées ou faisant l’objet de liens externes, le changement apporte peu : elles figurent de toute façon dans l’index via le crawler. Il est surtout utile pour les pages récentes qui ne figurent encore dans aucun index.

## Mesure : dans quelle mesure les langues sont-elles touchées ?

On peut mesurer si la règle touche les langues différemment en faisant passer les mêmes termes dans différentes langues à travers le code original de Brave. Les scripts, les données et les résultats individuels se trouvent dans le dépôt public [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung) ; la mesure peut y être reproduite avec cinq commandes.

**Méthode :**

- **Code :** dépôt de Brave `web-discovery-project`, commit `58b1b53` du 9 septembre 2026. Les fonctions `cleanCurrentUrl` et `dropLongURL` s’exécutent sans modification dans Node.js ; seules les connexions au navigateur et au stockage ont été remplacées. Les propres cas de test de Brave pour `dropLongURL` produisent ainsi les résultats attendus.
- **Termes :** la liste « Vital Articles Level 3 » de Wikipédia en anglais, soit environ 1 000 articles centraux. Les liens interlangues de Wikipédia ont permis de récupérer les titres des mêmes articles dans 23 autres langues. 825 termes existent dans les 24 langues. Pour chaque terme, seule la langue change.
- **URL :** `https://<sprache>.wikipedia.org/wiki/<Titel>`, formées comme dans le navigateur (encodées en pourcentage), puis passées comme dans le WDP d’abord par `cleanCurrentUrl`, puis par `dropLongURL` en mode normal et strict.
- **Cause :** chaque URL rejetée a été vérifiée une seconde fois, avec la règle de longueur désactivée. Si elle est alors signalée, le rejet est dû à la règle de longueur.
- **Statistiques :** intervalle de confiance à 95 % selon Wilson ; comparaison avec l’anglais sur les termes appariés (test exact de McNemar).

Le cœur de l’analyse :

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

**Résultat** (mode normal, 825 termes par langue) :

| Langue | Rejetés | IC à 95 % | dont règle de longueur | p par rapport à l’anglais |
|---|---|---|---|---|
| Anglais | 0 (0,0 %) | 0,0–0,5 % | – | – |
| Allemand | 10 (1,2 %) | 0,7–2,2 % | 10 | 0,002 |
| Hongrois | 9 (1,1 %) | 0,6–2,1 % | 6 | 0,004 |
| Néerlandais | 6 (0,7 %) | 0,3–1,6 % | 6 | 0,03 |
| Finnois | 5 (0,6 %) | 0,3–1,4 % | 5 | 0,06 |
| Suédois | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Norvégien | 4 (0,5 %) | 0,2–1,2 % | 4 | 0,13 |
| Danois, français, espagnol, portugais, turc, polonais | 1 chacun (0,1 %) | 0,0–0,7 % | 0–1 | 1,0 |
| Russe, ukrainien, japonais | 1 chacun (0,1 %) | 0,0–0,7 % | 1 | 1,0 |
| Italien, grec, arabe, hébreu, persan, hindi, chinois, coréen | 0 (0,0 %) | 0,0–0,5 % | – | – |

Ont par exemple été rejetés `Schwangerschaftsabbruch`, `Empfängnisverhütung` et `Ingenieurwissenschaften` (allemand), `Terhességmegszakítás` (hongrois), `Milieuverontreiniging` (néerlandais) et `Tietojenkäsittelytiede` (finnois). Les cas isolés dans les autres langues concernent presque tous le même terme : l’acide désoxyribonucléique dans la langue concernée. Dans aucun cas un terme n’a été signalé dans la langue locale et rejeté en anglais.

**Analyse :**

- Les langues utilisant une écriture non latine ne sont pas désavantagées. Brave décode l’URL avant la vérification ; un caractère cyrillique ou chinois compte donc comme un seul caractère.
- Les langues qui écrivent les mots composés en un seul bloc subissent un désavantage faible mais systématique. Pour l’allemand, il est significatif avec p = 0,002, même en tenant compte de la comparaison simultanée de 23 langues (seuil de Bonferroni 0,0022). Le hongrois se situe juste au-dessus.
- La proportion mesurée s’applique aux titres de Wikipédia, qui se composent généralement d’un ou deux mots. Les slugs de blogs contiennent davantage de termes techniques ; sur ce site, 3 URL d’articles allemands sur 66 sont concernées (4,5 %).

**Limites :** la règle de vérification a été mesurée, non l’inclusion effective dans l’index de Brave. Wikipédia elle-même figure probablement sur la liste d’autorisation de Brave et n’est pas réellement concernée ; la mesure montre comment la règle traite un site web sans autorisation avec de telles URL. Le commit public ne correspond pas nécessairement à la version livrée dans le navigateur. La mesure ne reproduit pas la voie de contournement via l’URL canonique.

## Avis : une limite de longueur qui désavantage les langues à mots composés

*Cette section présente l’évaluation de l’auteur. Les faits correspondants figurent dans les sections ci-dessus.*

La limite de 18 caractères compte les caractères et touche donc les langues qui assemblent les termes. La mesure montre l’effet : pour des termes identiques, la règle rejette 1,2 % des URL allemandes et 0 % des URL anglaises, et chacun de ces rejets allemands est dû à la règle de longueur. Le slug anglais `data-protection-regulation` passe la vérification, `datenschutzgrundverordnung` non. L’effet est faible, mais touche toujours les mêmes langues : allemand, hongrois, néerlandais, finnois et langues scandinaves. À mon sens, c’est un désavantage pour ces langues, même s’il n’est pas intentionnel.

### Qui supporte le coût

Brave écrit dans le README que les erreurs de classification ne sont pas un grand problème pour son propre objectif. C’est vrai pour Brave : une page rejetée coûte à Brave un signalement. Pour les sites web concernés, c’est un désavantage systématique qui touche surtout les textes spécialisés, car les termes techniques forment précisément de longs composés. Comme la recherche web de Claude repose sur l’index de Brave, ces pages perdent une voie vers l’index dont Claude tire ses citations.

S’ajoute une liste d’autorisation côté serveur : les modèles d’URL que Brave y inclut (`allowlisted`) ignorent la vérification. Les modèles concernés ne sont pas documentés publiquement. Les sites spécialisés isolés n’ont aucune influence là-dessus.

### Ce qui plaide en faveur de la règle

L’objectif est légitime. Les liens de partage avec jeton, par exemple pour des documents partagés, représentent un risque réel, et un tel lien dans l’index de recherche serait un grave incident de protection des données. Une limite de longueur stricte est simple, rapide et difficile à contourner. Elle ne peut pas non plus être simplement remplacée par la détection de hash : dans un test complémentaire avec 2 000 jetons aléatoires de 22 caractères en minuscules, `isHash` de Brave n’en a classé que 58 % comme hash ; pour les jetons composés de minuscules et de chiffres, ce taux était de 94 %. Tous les composés testés tels que `schwangerschaftsabbruch` ou `datenschutzgrundverordnung` ont été correctement reconnus par la fonction comme n’étant pas des hash. Pour les liens secrets constitués uniquement de minuscules, la limite de longueur est donc la véritable protection. De plus, le crawler continue d’atteindre les pages concernées, et la proportion mesurée est faible.

### Ce que Brave pourrait changer

- **Distinguer les mots des jetons :** relever simplement la limite pour les mots composés uniquement de lettres laisserait passer les liens secrets en minuscules, selon le test complémentaire. Une vérification de plausibilité supplémentaire uniquement pour les parties de mots entre 19 et environ 30 caractères serait envisageable, par exemple via la succession des voyelles et consonnes ou un petit modèle linguistique entraîné sur des mots des langues concernées. Brave devrait vérifier si cela est suffisamment fiable.
- **Documenter la règle :** la page d’aide consacrée au crawler mentionne le WDP, mais pas les heuristiques. Une indication pour les exploitants de sites web suffirait pour qu’ils puissent adapter leurs slugs en conséquence.

J’ai signalé la mesure et ce conflit d’objectifs à Brave dans l’[Issue #498](https://github.com/brave/web-discovery-project/issues/498) du dépôt du WDP. En attendant, il ne reste que l’adaptation du côté du site web : séparer les composés dans le slug par des traits d’union. Le fait que ce travail incombe aux exploitants de certaines régions linguistiques plutôt qu’au procédé lui-même est le cœur de ma critique.

*Correction du 25.09.2026 : une version antérieure proposait de relever de manière générale la limite pour les mots composés uniquement de lettres ; le test complémentaire sur la détection de hash montre que cela affaiblirait la protection. Une version antérieure de cette section affirmait que les écritures non latines et les diacritiques étaient particulièrement désavantagés par l’encodage en pourcentage, sur la base d’une mesure avec des URL encodées en pourcentage. Une vérification indépendante a montré que Brave décode les URL avec `cleanCurrentUrl` avant la vérification. Cette affirmation était erronée et a été supprimée ; la mesure ci-dessus utilise le déroulement correct.*

## Sources

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md) : description du WDP par Brave, avec les types de messages, la seconde consultation sans cookies, les heuristiques d’URL à capacité et le quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es) : code source avec `dropLongURL`, `calculateStrictness` et les seuils `rel_part_len: 18` et `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler) : page d’aide officielle sur le crawler, l’absence de user-agent propre et le rôle du WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/) : ajout de Brave Search à la liste des sous-traitants d’Anthropic en mars 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/) : article apportant d’autres indices, tel que le paramètre `BraveSearchParams` dans la recherche web de Claude.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226) : décodage de l’URL avant la vérification ; le commentaire dans onLocationChange (ligne 1724) décrit l’URL décodée comme la représentation interne du WDP.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3) : liste de termes de la mesure ; les titres dans les autres langues proviennent des liens interlangues de l’API Wikipédia.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung) : scripts de mesure, jeu de données et résultats individuels de la mesure linguistique de cet article, y compris le test complémentaire de détection de hash (`hash-test.mjs`) et la première version signalée comme erronée.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498) : retour adressé à Brave avec les résultats de la mesure et le conflit d’objectifs entre limite de longueur et détection de hash.
