---
title: "The Attacked Pension Fund Service Provider Has No DMARC Record at All"
navTitle: "DMARC Pension Funds"
description: "PK Softech, the pension fund service provider attacked at the end of September 2026, still has no DMARC record. A passive review of 66 pension institutions and 30 service providers shows that this is not an isolated case: the vendors of administrative software in particular are unprotected."
date: "2026-10-18"
kategorie: "SMTP and Mail Flow"
timeToRead: "9 min to read"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 56251be16ac00d877c2536fc6384a564114736e38301637d9006f8e21da77d9a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T08:32:11.603Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all
---

At the end of September 2026, PK Softech AG was attacked, a software vendor whose pension solution is used by several Swiss pension funds. Data was exfiltrated through this route. According to reporting, those affected include the federal pension fund Publica, Migros Pension Fund, and the Pension Fund of the City of Zurich, as well as other institutions. Depending on the fund, this involves names, AHV numbers, marital status, addresses, and IBANs, as well as salary and pension data.

This exfiltration is serious in itself. It affects identification and financial data for a large part of the working population and, through Publica, large parts of the federal administration. Data of this volume and quality is not merely a problem for the individual. It enables identity abuse, targeted deception, and extortion on a large scale, making it a threat to public order and national security. The pension assets themselves remained unaffected, but the information platform Cybercrimepolice.ch already warned about fraudulent calls using the stolen data.

There is also a finding that is difficult to explain for a service provider in this category: **PK Softech AG still has no DMARC record.** Its domain has a strict SPF record but no DMARC policy (checked on October 18, 2026). This makes it possible to spoof the attacked service provider's sender address without a receiving mail platform reliably rejecting it. After a data exfiltration, this is the worst conceivable starting point. Anyone who knows an insured person's name, address, and bank details and can also spoof an email in the name of the service provider or fund can write a message that is barely distinguishable from a genuine one, such as a request to update the IBAN for the next pension payment. DMARC protects against exactly this kind of spoofing.

PK Softech is not an isolated case, but the clearest example of a pattern. I reviewed how well Swiss pension institutions and their typical service providers are positioned for email authentication.

## How the review was conducted

The evaluation covers 66 second-pillar pension institutions, from large public-law funds and insurers' collective foundations to smaller joint institutions. Only public DNS records were reviewed: SPF, DKIM, DMARC, MX records, as well as DNSSEC and DANE (TLSA records of the mail servers, checked on October 19, 2026). This is the same information that every receiving mail system queries for every incoming message.

The review was purely passive. Only public DNS was queried; no connection was made to the institutions' systems, no login was attempted, and no personal data was collected. The evaluation reflects only what can already be read from public sources by anyone. Corporate collective foundations are consolidated under the sponsor's primary domain; individual affiliated foundations may differ.

## The results

| Feature | Share of the 66 institutions |
|---|---|
| SPF present | 88% |
| SPF strict (`-all`) | 59% |
| DMARC present | 59% |
| DMARC `p=reject` (full protection) | 20% |
| DMARC `p=quarantine` | 26% |
| DMARC `p=none` (monitoring only) | 14% |
| no DMARC | 41% |
| DKIM detectable | 55% |
| DNSSEC | 38% |
| DANE | 9% |

Three points stand out.

**Only 20 percent are effectively protected with DMARC `p=reject`.** Only this policy instructs a receiving mail platform to actually reject spoofed messages. In all other cases, a spoofed message will likely end up in the insured person's inbox.

**41 percent have no DMARC.** Their sender domain can readily be used for fraudulent emails, and the institution will not even know about it because DMARC's reporting function is also absent without DMARC. Notably, this includes funds that run their email on Microsoft 365, where DMARC can be configured with little effort.

**DMARC `p=none` is not protection.** The 14 percent in this group have set a DMARC policy but left it at the monitoring stage. This provides reports but does not reject a single spoofed message. In practice, it is an interim step that often remains in place for months or years.

Transport encryption shows a similar picture: only 6 of the 66 institutions (9 percent) protect their mail servers with DANE. This allows a sending mail server to verify, using a DNSSEC-signed record, that it is connected securely to the correct recipient. DNSSEC is a prerequisite, and 38 percent of institutions have already enabled it. For them, DANE is the logical next step.

## Browse all datasets

The following infographic shows the reviewed status of all 66 pension institutions and 30 service providers. The filters can be used to narrow results by DMARC status, category, and mail provider, and each entry can be expanded for the complete DNS findings.

<div data-dmarc-infographic></div>

## For comparison: Switzerland's largest companies

The [Swiss Email Security Index](/open-data/mail-security-index) shows how pension institutions compare with the rest of the economy. Using the same passive method, it reviews more than 500 of Switzerland's largest companies daily, with an interactive map by canton, comparisons by industry and mail provider, and a detail page for each company. This also includes the banks, insurers, and IT service providers that pension institutions work with.

The gap is clear: of the 518 companies in the index, 38 percent use DMARC with `p=reject` as of October 18, 2026, compared with 20 percent of pension institutions. Just 5 percent of companies have no DMARC at all, compared with 41 percent of pension institutions.

## Suppliers are the bigger issue

The incident at the end of September affected not a fund, but a shared service provider. I therefore reviewed 30 typical suppliers using the same passive method: vendors of administrative software, companies providing outsourced management, actuarial and consulting services, custodian banks, and IT service providers.

On average, service providers are somewhat better positioned than the funds: 40 percent use DMARC with `p=reject`, compared with 20 percent of the institutions. What matters, however, is the distribution across categories.

| Category | with DMARC `p=reject` |
|---|---|
| Actuarial and consulting | good |
| Custodian banks | good |
| IT service providers | mixed |
| Outsourced management | weak |
| Administrative software | weakest |

Vendors of administrative software are the weakest group. None of the reviewed software providers use DMARC with full effect, and the attacked vendor PK Softech has no DMARC record at all. This is precisely the category that processes pension data and is deeply connected to the funds through interfaces, and it is the category involved in the current incident.

This leads to the central point regarding due diligence: a fund with `p=reject` is well protected as long as the attacker wants to spoof its domain. But if the attacker can instead spoof the domain of a poorly protected service provider through which communication with insured persons already takes place, the email is just as credible. Securing one's own domain is therefore only the first step. The second is requiring the most important service providers to meet the same standard, ideally by contract.

## What this means for the board of trustees

Responsibility for information security lies with the pension institution's supreme governing body. A data exfiltration also triggers specific obligations under the revised Data Protection Act: a security breach must be reported to the FDPIC if it is likely to result in a high risk for the affected persons, and those affected must be informed where necessary for their protection.

In this situation, email authentication is the most effective individual measure with the least effort. SPF, DKIM, and DMARC require no license and no new software. They consist of DNS records and a clean configuration of the existing mail platform. Protection takes effect immediately for every insured person, without requiring them to do anything themselves.

## What needs to be done

For institutions without effective protection, a clear path leads to the goal:

1. **Review SPF and set it strictly.** The SPF record lists authorized sending systems and ends with `-all`, so that all other senders are considered unauthorized.
2. **Enable DKIM.** Every outgoing message is cryptographically signed. In Microsoft 365 and common mail gateways, this is a setting, not a project.
3. **Introduce DMARC and tighten it gradually.** Start with `p=none` and active reporting to capture legitimate mail traffic. Then `p=quarantine`, and finally `p=reject`. Depending on the environment, the entire process takes a few weeks, not months.
4. **Evaluate reporting.** DMARC reports show who sends email in the domain's name and uncover both forgotten legitimate systems and abuse.
5. **At least assess DANE and review mail encryption.** DANE secures transport encryption between mail servers through DNSSEC. Whether the organization's own mail platform and DNS provider support it can be clarified with little effort. For messages containing salary and pension data, it should also be defined how they are sent end-to-end encrypted, for example through a mail gateway with S/MIME or secure delivery via a web portal.
6. **Hold service providers accountable.** Software vendors, outsourced management providers, and IT partners should demonstrate the same level of SPF, DKIM, and DMARC. This belongs in contracts and regular supplier reviews.

## Conclusion

The recent attacks targeted a service provider, not the funds themselves. But the resulting damage occurs wherever a sender domain is unprotected, whether at the fund or among its suppliers. Four out of five reviewed pension institutions are not fully protected against spoofed senders, two out of five have no DMARC at all, and vendors of administrative software in particular perform worst. The gap is known, inexpensive to close, and effective immediately. After a data exfiltration involving names, addresses, and bank details, it is not a minor technical matter, but part of the supreme governing body's duty of care throughout the entire supply chain.

## Sources

1.  [SRF: Federal pension fund, how could sensitive data end up at an external software company?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Background on the incident, the types of data affected, and the role of the software supplier.

2.  [cash.ch (AWP): Attack on Publica supplier, salary data may have been affected](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifies PK Softech AG to AWP as the company attacked.

3.  [Netzwoche: Federal pension fund Publica reports data exfiltration](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Report on the attack on the software supplier and the data exfiltration.

4.  [Mimikama: Swiss police warn of possible investment calls after data exfiltration](https://www.mimikama.org/?p=436628): Context for the warning about fraudulent calls using the exfiltrated data.

5.  [FDPIC: Reporting security breaches](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Reporting obligation under the revised Data Protection Act and procedures for data security breaches.
