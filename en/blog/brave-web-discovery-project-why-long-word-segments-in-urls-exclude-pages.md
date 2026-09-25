---
title: "Brave Web Discovery Project: Why Long Word Segments in URLs Exclude Pages"
navTitle: "Brave: long URL segments"
description: "In the Web Discovery Project, Brave discards any URL whose path contains a word segment longer than 18 characters. German compounds such as Verschlüsselungsgateway are affected. Because Claude's web search is based on the Brave index, this also affects visibility in Claude. The source-code rule, further heuristics, the role of the canonical URL, a script for checking your own sitemap, a measurement using 825 terms in 24 languages, and an assessment of why the rule disadvantages compound-word languages."
date: "2026-09-25"
kategorie: "Claude"
timeToRead: "13 min read"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "troubleshooting"
slug: "brave-web-discovery-project-why-long-word-segments-in-urls-exclude-pages"
translationId: "article-2cf5ffd0d1886c78"
aiPrompt: |
  Du bist mein SEO-Assistent. Hilf mir zu prüfen, ob die URLs meiner Website die Heuristiken des Brave Web Discovery Project (dropLongURL) bestehen: Wortteile im Pfad über 18 Zeichen, lange Query-Strings, lange Zahlen, Pfade wie /admin oder /share. Werte meine Sitemap aus, nenne die betroffenen URLs und schlage kürzere Slugs mit 301-Weiterleitung vor, wo sich die Änderung lohnt.
translationOf: brave-wdp-lange-url-wortteile
url: https://rafaelpfister.ch/en/blog/brave-web-discovery-project-why-long-word-segments-in-urls-exclude-pages
translationSourceHash: a035c348521de8ab826dc9f47abbad518f9bf4d07c0ef6d99e6796a7b85afa67
translationModel: gpt-5.6-terra
translatedAt: 2026-09-25T08:56:04.103Z
translationReview: required
---

# Brave Web Discovery Project: Why Long Word Segments in URLs Exclude Pages

Claude's web search delivers its results from the Brave Search index. Anthropic has listed Brave Search as a subprocesser since March 2025, and citations in Claude responses largely match Brave results. Anyone who wants to be cited in Claude responses must therefore be in the Brave index. Submissions through Google Search Console, Bing Webmaster Tools, or IndexNow do not reach Brave directly.

Brave fills its index from two sources: its own crawler and the Web Discovery Project (WDP). The WDP is an opt-in feature in the Brave browser that anonymously reports visited pages to Brave. Before the browser reports a page, it checks the URL using a series of heuristics. One of them affects German-language websites in particular: If the path contains a word segment longer than 18 characters, the page is never reported.

## What the Web Discovery Project reports

The WDP collects two types of data: search queries on Google, Bing, Brave, and several other search engines along with their result lists, and page visits with URL, title, dwell time, and interaction. There are no user IDs; each report is sent individually, encrypted, and at randomly distributed times to Brave.

To prevent private content from being reported, the browser classifies a page as private if

- a second request without cookies returns a substantially different page (login, personalization),
- the domain points to a private IP address,
- the page carries `noindex`,
- the URL looks like a secret link (a capability URL, such as a sharing link with a token).

The last check is handled by the function `dropLongURL`. The browser permanently stores a URL classified as private in a Bloom filter and does not check it again.

## The rule: no word segment over 18 characters

`dropLongURL` splits the path at slashes, periods, underscores, spaces, hyphens, colons, plus signs, and semicolons, then compares each segment with the threshold `rel_part_len`:

```javascript
var vpath = url_parts.path.split(/[\/\._ \-:\+;]/);
for (var i = 0; i < vpath.length; i++) {
  if (vpath[i].length > WebDiscoveryProject.rel_part_len) {
    return true;
  }
}
```

`rel_part_len` is set to `18` in the source code. `return true` means: the URL is considered suspicious. The decoded URL is checked: The browser percent-encodes non-ASCII characters (`%C3%BC`), and the WDP converts them back beforehand using `cleanCurrentUrl` (`decodeURIComponent`). JavaScript characters are counted, i.e., UTF-16 code units: An `ü` or a Chinese character counts as one, while a character outside the Unicode Basic Multilingual Plane or an accent appended as a separate character counts twice. The rule applies in both check modes, normal and strict. Brave itself describes the heuristics in the README as conservative: Many public pages are falsely classified as possible secret links, which is accepted for the WDP's purpose.

A hyphen separates, but a concatenated word does not. What matters is therefore the length of the longest individual word in the slug, not the length of the entire slug:

| Slug | Longest segment | Result |
|---|---|---|
| `microsoft-graph-powershell-postfach-anbindung` | `powershell` (10) | is reported |
| `hin-plattformerneuerung-2026` | `plattformerneuerung` (19) | discarded |
| `verschluesselungsgateway-hinter-exchange-online` | `verschluesselungsgateway` (24) | discarded |
| `verschluesselungs-gateway-hinter-exchange-online` | `verschluesselungs` (17) | is reported |

## Why German slugs are particularly affected

English technical terms are written separately, while German ones are compounded. There is also the transliteration of umlauts: `ü` becomes `ue`, making every word with an umlaut longer. `Zertifikatserneuerung` has 21 characters, `Verschlüsselungsgateway` has 24 in ASCII transliteration. Scandinavian languages are similar: Swedish and Norwegian also concatenate compounds.

On this website, an analysis of the sitemap found 24 affected URLs out of 750: three German articles, one English article, and 20 Swedish or Norwegian translations. The English case comes from the header field `MessageDirectionality`, which was carried over into the slug as a word.

## When the canonical URL helps

If the requested URL fails `dropLongURL`, the browser checks the page's canonical URL. If it passes the check, it reports the page under the canonical URL. This covers the typical case where a page is requested with tracking parameters but has a clean canonical URL.

It does not help with a word segment that is too long in the slug: The canonical URL is generally identical to the requested URL and fails the same rule. If both fail, the browser discards the page.

Whether strict check mode applies also depends on the canonical URL: It is relaxed only if the canonical differs from the requested URL and is shorter. A page that identifies itself as canonical is subject to strict mode.

## Further heuristics in dropLongURL

In addition to word-segment length, `dropLongURL` discards a URL, among other things, when it has

- a query string longer than 30 characters or with more than four parameters (in strict mode, from 23 characters or two parameters),
- a digit sequence of more than 12 digits in the path or query string (more than 8 in strict mode); special characters are removed first, so a date path such as `/2026/09/25/` counts as an eight-digit number,
- word segments that a Markov classifier classifies as a hash,
- path segments such as `/admin`, `/wp-admin`, `/edit`, `/share`, `/logout` or `/token`,
- an email address in the URL.

For a typical article URL without parameters, only word length is usually relevant.

## Context: quorum and crawler

The rule affects only the WDP route. Two points limit its significance:

- **Quorum:** Brave can decrypt a page report only after more than a certain number of users have reported the same URL within 30 days from different networks. The README does not state the threshold. For pages with few Brave visitors, the WDP therefore has little effect anyway.
- **Crawler:** The Brave crawler operates independently of the WDP. It has no dedicated user agent and follows the robots.txt rules for Googlebot. A URL with a long word segment can still enter the index through the crawler.

A long word segment thus removes one of the two routes into the Brave index; the crawler route remains available.

## Check your own URLs

The following commands read a sitemap and output every URL whose path contains a segment longer than 18 characters. For a sitemap index, check the individual sitemaps (`sitemap-0.xml` etc.) one after another.

On Linux or macOS:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `curl -s` | Downloads the sitemap without a progress indicator. |
| `grep -oE '<loc>[^<]+'` | Outputs only the `<loc>` entries (`-o`), using extended regular expressions (`-E`). |
| `sed -E 's#…##'` | Removes `<loc>`, the scheme, and hostname; the path remains. |
| `awk -F'[/._ :+;-]'` | Splits the path at the same separators as `dropLongURL`. |
| `length($i) > 18` | Outputs the length, word segment, and path as soon as a segment exceeds the threshold. |

</details>

On Windows with PowerShell:

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
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Invoke-RestMethod -Uri` | Downloads the sitemap and returns it directly as an XML object. |
| `$sitemap.urlset.url.loc` | Reads all `<loc>` entries from the XML. |
| `([uri]$loc).AbsolutePath` | Removes the scheme and hostname and returns the path. |
| `[uri]::UnescapeDataString()` | Decodes percent encoding, just as the WDP does before checking. |
| `-split '[/._ :+;-]'` | Splits the path at the same separators as `dropLongURL`. |
| `Where-Object { $_.Length -gt 18 }` | Keeps only word segments longer than 18 characters. |

</details>

Both variants check only word length, not the other heuristics. The Bash variant counts the percent-encoded form and therefore produces correct values only for purely ASCII slugs; use the PowerShell variant for slugs with umlauts or other scripts.

## Adjust slugs

For new articles, the rule can be observed when defining the slug: Separate compounds in the slug with hyphens (`verschluesselungs-gateway`, `plattform-erneuerung`, `zertifikats-erneuerung`) and shorten identifiers taken from technical terms.

For existing URLs, a change should be weighed carefully. Every slug change requires a 301 redirect from the old URL to the new one, an updated sitemap, and adjusted internal links. For pages that already rank well or have external links, the change offers little benefit: They are already in the index through the crawler. It is mainly useful for young pages that are not yet in any index.

## Measurement: How much are individual languages affected?

Whether the rule affects languages differently can be measured by running the same terms in different languages through Brave's original code. Scripts, data, and individual results are available in the public repository [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung); the measurement can be reproduced there with five commands.

**Setup:**

- **Code:** Brave's repository `web-discovery-project`, commit `58b1b53` of September 9, 2026. The functions `cleanCurrentUrl` and `dropLongURL` run unchanged in Node.js; only the connections to browser and storage were replaced. Brave's own test cases for `dropLongURL` therefore produce the expected results.
- **Terms:** the English Wikipedia list “Vital Articles Level 3,” around 1,000 key articles. Wikipedia language links were used to retrieve the titles of the same articles in 23 additional languages. 825 terms exist in all 24 languages. Thus, only the language changes for each term.
- **URLs:** `https://<sprache>.wikipedia.org/wiki/<Titel>`, constructed as in the browser (percent-encoded), then passed through `cleanCurrentUrl` first, as in the WDP, and then through `dropLongURL` in normal and strict mode.
- **Cause:** Every discarded URL was checked a second time with the length rule disabled. If it was then reported, the discard was attributable to the length rule.
- **Statistics:** 95% Wilson confidence interval; comparison with English using paired terms (exact McNemar test).

The core of the analysis:

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

**Result** (normal mode, 825 terms per language):

| Language | Discarded | 95% CI | of which length rule | p vs. English |
|---|---|---|---|---|
| English | 0 (0.0%) | 0.0–0.5% | – | – |
| German | 10 (1.2%) | 0.7–2.2% | 10 | 0.002 |
| Hungarian | 9 (1.1%) | 0.6–2.1% | 6 | 0.004 |
| Dutch | 6 (0.7%) | 0.3–1.6% | 6 | 0.03 |
| Finnish | 5 (0.6%) | 0.3–1.4% | 5 | 0.06 |
| Swedish | 4 (0.5%) | 0.2–1.2% | 4 | 0.13 |
| Norwegian | 4 (0.5%) | 0.2–1.2% | 4 | 0.13 |
| Danish, French, Spanish, Portuguese, Turkish, Polish | 1 each (0.1%) | 0.0–0.7% | 0–1 | 1.0 |
| Russian, Ukrainian, Japanese | 1 each (0.1%) | 0.0–0.7% | 1 | 1.0 |
| Italian, Greek, Arabic, Hebrew, Persian, Hindi, Chinese, Korean | 0 (0.0%) | 0.0–0.5% | – | – |

Examples of discarded terms include `Schwangerschaftsabbruch`, `Empfängnisverhütung` and `Ingenieurwissenschaften` (German), `Terhességmegszakítás` (Hungarian), `Milieuverontreiniging` (Dutch), and `Tietojenkäsittelytiede` (Finnish). The individual cases in the other languages almost all concern the same term: deoxyribonucleic acid in the respective local language. In no case was a term reported in the local language and discarded in English.

**Analysis:**

- Languages using non-Latin scripts are not disadvantaged. Brave decodes the URL before checking, so a Cyrillic or Chinese character counts as one character.
- Languages that concatenate compounds have a small but systematic disadvantage. For German, it is significant at p = 0.002 even after accounting for the simultaneous comparison of 23 languages (Bonferroni threshold 0.0022). Hungarian is just above that threshold.
- The measured share applies to Wikipedia titles, which usually consist of one or two words. Blog slugs contain more technical terms; on this website, 3 of 66 German article URLs are affected (4.5%).

**Limitations:** The check rule was measured, not actual inclusion in the Brave index. Wikipedia itself is probably on Brave's allowlist and is not affected in practice; the measurement shows how the rule treats a website without an exception that has such URLs. The public commit may not correspond to the version delivered in the browser. The measurement does not model the fallback route through the canonical URL.

## Opinion: A length limit that disadvantages compound-word languages

*This section reflects the author's assessment. The relevant facts appear in the sections above.*

The 18-character limit counts characters and therefore affects languages that concatenate terms. The measurement shows the effect: For identical terms, the rule discards 1.2% of German URLs and 0% of English URLs, and every one of those German discards is due to the length rule. The English slug `data-protection-regulation` passes the check, while `datenschutzgrundverordnung` does not. The effect is small, but it always affects the same languages: German, Hungarian, Dutch, Finnish, and the Scandinavian languages. In my view, this disadvantages these languages, even if it is unintentional.

### Who bears the costs

Brave writes in the README that false classifications are not a major problem for its own purpose. That is true for Brave: A discarded page costs Brave one report. For affected websites, it is a systematic disadvantage that primarily affects technical texts, because technical terms in particular form long compounds. Since Claude's web search is based on the Brave index, these pages lose one route into the index from which Claude cites.

There is also a server-side allowlist: URL patterns that Brave includes there (`allowlisted`) skip the check. Which patterns those are is not publicly documented. Individual specialist websites have no influence over this.

### What supports the rule

The purpose is legitimate. Sharing links with tokens, for example for shared documents, are a real risk, and such a link in a search index would be a serious privacy incident. A hard length limit is simple, fast, and difficult to circumvent. Nor can it simply be replaced with hash detection: In an additional test with 2,000 random 22-character lowercase tokens, Brave's `isHash` classified only 58% as hashes; for tokens containing lowercase letters and digits, it classified 94%. The function correctly recognized all tested compounds such as `schwangerschaftsabbruch` or `datenschutzgrundverordnung` as not being hashes. For secret links made up solely of lowercase letters, the length limit is therefore the actual protection. In addition, the crawler can still reach affected pages, and the measured share is small.

### What Brave could change

- **Distinguish words from tokens:** Simply raising the limit for words consisting only of letters would, according to the additional test, let secret links made of lowercase letters through. An additional plausibility check only for word segments between 19 and about 30 characters could be conceivable, for example based on the sequence of vowels and consonants or a small language model trained on words from the affected languages. Brave would need to verify whether this is reliable enough.
- **Document the rule:** The crawler help page mentions the WDP but not the heuristics. A note for website operators would be enough to let them adapt their slugs accordingly.

I reported the measurement and this trade-off to Brave as [Issue #498](https://github.com/brave/web-discovery-project/issues/498) in the WDP repository. Until then, the only option is to adapt things on the website side: Separate compounds in the slug with hyphens. The fact that this work falls on operators in certain language regions rather than on the method itself is the core of my criticism.

*Correction dated September 25, 2026: An earlier version proposed generally raising the limit for words consisting only of letters; the additional hash-detection test shows that this would weaken the protection. An earlier version of this section claimed that non-Latin scripts and diacritics were particularly disadvantaged by percent encoding, based on a measurement with percent-encoded URLs. An independent review showed that Brave decodes URLs using `cleanCurrentUrl` before checking. The claim was incorrect and has been removed; the measurement above uses the correct process.*

## Sources

1.  [brave/web-discovery-project: sources/README.md](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/README.md): Brave's description of the WDP, including message types, second request without cookies, capability-URL heuristics, and quorum.

2.  [brave/web-discovery-project: web-discovery-project.es](https://github.com/brave/web-discovery-project/blob/main/modules/web-discovery-project/sources/web-discovery-project.es): Source code containing `dropLongURL`, `calculateStrictness` and the thresholds `rel_part_len: 18` and `qs_len: 30`.

3.  [Brave Search: Brave Search Crawler](https://search.brave.com/help/brave-search-crawler): Official crawler help page, on the lack of a dedicated user agent and the role of the WDP.

4.  [Simon Willison: Anthropic Trust Center: Brave Search added as a subprocessor](https://simonwillison.net/2025/Mar/21/anthropic-use-brave/): Entry for Brave Search in Anthropic's list of subprocessors in March 2025.

5.  [TechCrunch: Anthropic appears to be using Brave to power web searches for its Claude chatbot](https://techcrunch.com/2025/03/21/anthropic-appears-to-be-using-brave-to-power-web-searches-for-its-claude-chatbot/): Report with further evidence, such as the parameter `BraveSearchParams` in Claude's web search.

6.  [brave/web-discovery-project: cleanCurrentUrl](https://github.com/brave/web-discovery-project/blob/58b1b53f046e955d9d577ac531d7d6b4d18a6016/modules/web-discovery-project/sources/web-discovery-project.es#L2226): URL decoding before checking; the comment in onLocationChange (line 1724) describes the decoded URL as the WDP's internal representation.

7.  [Wikipedia: Vital articles/Level 3](https://en.wikipedia.org/wiki/Wikipedia:Vital_articles/Level_3): Term list for the measurement; titles in the other languages come from Wikipedia API language links.

8.  [pfstr/wdp-sprachmessung](https://github.com/pfstr/wdp-sprachmessung): Measurement scripts, dataset, and individual results of the language measurement in this article, including the additional hash-detection test (`hash-test.mjs`) and the first version marked as incorrect.

9.  [brave/web-discovery-project#498](https://github.com/brave/web-discovery-project/issues/498): Feedback to Brave with the measurement results and the trade-off between the length limit and hash detection.
