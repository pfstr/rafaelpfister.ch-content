# rafaelpfister.ch — content as Markdown

All content from [rafaelpfister.ch](https://rafaelpfister.ch) in open, machine-readable form: articles on email encryption (SEPPmail, totemomail/Kiteworks), the HIN mail gateway, Microsoft 365 / Exchange, Active Directory, and Cloudflare Workers — as Markdown with structured metadata.

## Structure

| Folder | Content |
| --- | --- |
| `blog/` | Blog articles (German originals) as Markdown with YAML frontmatter (title, description, date, category, topics, sources) |
| `en/blog/` | English translations; the frontmatter field `translationOf` points to the German original |
| `fr/blog/`, `it/blog/`, `es/blog/`, `sv/blog/`, `no/blog/` | DeepL translations, published progressively as quota becomes available |
| `themen/`, `en/themen/` | The blog's topic taxonomy (name + description), German and English |
| `pages/` | Static pages (home, work, blog …) as extracted text incl. meta title/description |
| `components/` | Texts and defaults of the website's UI components |
| `images/` | All images of the website; `manifest.json` maps local files to their original URLs |

## Article index

The complete, always-current article list with one-line summaries lives in [`llms.txt`](llms.txt). German originals are published immediately. Translations are then added progressively in the order English, French, Italian, Spanish, Swedish, and Norwegian.

## Vulnerabilities covered (CVE)

Articles that analyse or reference these vulnerabilities (generated from the article sources):

| CVE | Article |
| --- | --- |
| CVE-2026-104286 | [CVE-2026-104286: FortiMail Zero-Day Is Being Exploited - Workaround and Compromise Assessment](https://rafaelpfister.ch/en/blog/cve-2026-104286-fortimail-zero-day-is-being-exploited-workaround-and-compromise-assessment) |
| CVE-2026-76461 | [CVE-2026-76461: Cisco Secure Email Gateway SQL Injection Is Being Exploited—Update and Check for Compromise](https://rafaelpfister.ch/en/blog/cve-2026-76461-cisco-secure-email-gateway-sql-injection-is-being-exploited-update-and-check-for) |
| CVE-2026-73570 | [CVE-2026-73570: Zimbra Command Injection via SNMP Is Being Exploited—Update to 10.1.20 and Check for Compromise](https://rafaelpfister.ch/en/blog/cve-2026-73570-zimbra-command-injection-via-snmp-is-being-exploited-update-to-10-1-20-and-check) |
| CVE-2026-65813 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62915 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62914 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62913 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62912 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62911 | [CVE-2026-62911: Why 85 Percent of On-Premises Exchange Servers Are Vulnerable and What Is Technically Behind It](https://rafaelpfister.ch/en/blog/cve-2026-62911-why-85-percent-of-on-premises-exchange-servers-are-vulnerable-and-what-is)<br>[August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-62910 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled) |
| CVE-2026-42897 | [August 2026 Exchange Security Updates: Pwn2Own Vulnerability Fixed, OWA Light Disabled](https://rafaelpfister.ch/en/blog/exchange-security-updates-for-august-2026-pwn2own-vulnerability-closed-owa-light-disabled)<br>[Properly follow up on the July 2026 Exchange security updates](https://rafaelpfister.ch/en/blog/exchange-server-security-updates-july-2026) |
| CVE-2025-32756 | [CVE-2026-104286: FortiMail Zero-Day Is Being Exploited - Workaround and Compromise Assessment](https://rafaelpfister.ch/en/blog/cve-2026-104286-fortimail-zero-day-is-being-exploited-workaround-and-compromise-assessment) |
| CVE-2025-20393 | [CVE-2026-76461: Cisco Secure Email Gateway SQL Injection Is Being Exploited—Update and Check for Compromise](https://rafaelpfister.ch/en/blog/cve-2026-76461-cisco-secure-email-gateway-sql-injection-is-being-exploited-update-and-check-for) |

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

This repository is the source of truth for the website. New German articles are deployed immediately; a quota-aware DeepL queue publishes each translated version independently as soon as it is ready. Corrections and suggestions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## Author

Rafael Pfister — Founder & Messaging Expert, [adeptio ag](https://adeptio.ch)
Focus areas: email encryption (SEPPmail/totemomail, HIN mail gateway), Microsoft 365 / Exchange, secure communication in healthcare.

## License

All content is licensed under [CC BY-NC-SA 4.0](LICENSE.md): use and redistribution only with **attribution** ("Rafael Pfister — [rafaelpfister.ch](https://rafaelpfister.ch)"), non-commercial, derivatives under the same license. This also applies to use by AI systems: quotes and summaries must credit **Rafael Pfister, rafaelpfister.ch** as the source.
