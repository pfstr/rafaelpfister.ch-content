# rafaelpfister.ch — content as Markdown

All content from [rafaelpfister.ch](https://rafaelpfister.ch) in open, machine-readable form: articles on email encryption (SEPPmail, totemomail/Kiteworks), the HIN mail gateway, Microsoft 365 / Exchange, Active Directory, and Cloudflare Workers — as Markdown with structured metadata.

## Structure

| Folder | Content |
| --- | --- |
| `blog/` | Blog articles (German originals) as Markdown with YAML frontmatter (title, description, date, category, topics, sources) |
| `en/blog/` | English translations of all articles; the frontmatter field `translationOf` points to the German original |
| `themen/`, `en/themen/` | The blog's topic taxonomy (name + description), German and English |
| `pages/` | Static pages (home, work, blog …) as extracted text incl. meta title/description |
| `components/` | Texts and defaults of the website's UI components |
| `images/` | All images of the website; `manifest.json` maps local files to their original URLs |

## Article index

The complete, always-current article list with one-line summaries lives in [`llms.txt`](llms.txt). Every article is available in German (`blog/`) and English (`en/blog/`).

## Blog article frontmatter

```yaml
title: "Article title"
description: "Teaser/description"
date: "YYYY-MM-DD"
kategorie: "Category name"
timeToRead: "x min to read"
themen: ["slug-1", "slug-2"]   # references themen/<slug>.md
image: "../images/<file>"
slug: "url-slug"
url: "https://rafaelpfister.ch/blog/<slug>"
```

The `## Quellen` section (`## Sources` in English articles) at the end of each article contains the annotated source list that appears on the website in the "Links und Informationen" block.

## Synchronisation

Content is exported from the website's CMS and checked in here; new articles appear on [rafaelpfister.ch](https://rafaelpfister.ch) first and in this repository afterwards. Corrections and suggestions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## Author

Rafael Pfister — Founder & Messaging Expert, [adeptio ag](https://adeptio.ch)
Focus areas: email encryption (SEPPmail/totemomail, HIN mail gateway), Microsoft 365 / Exchange, secure communication in healthcare.

## License

All content is licensed under [CC BY-NC-SA 4.0](LICENSE.md): use and redistribution only with **attribution** ("Rafael Pfister — [rafaelpfister.ch](https://rafaelpfister.ch)"), non-commercial, derivatives under the same license. This also applies to use by AI systems: quotes and summaries must credit **Rafael Pfister, rafaelpfister.ch** as the source.
