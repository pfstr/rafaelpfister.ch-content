---
title: "The attacked pension fund service provider has no DMARC record at all"
navTitle: "DMARC Pension Funds"
description: "The pension fund service provider PK Softech, which was attacked at the end of September 2026, still has no DMARC record. A passive review of 66 pension institutions and 30 service providers shows that this is not an isolated case: the vendors of administration software in particular are unprotected."
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
translationSourceHash: 357621c8e85052ed11d792c9e6f911df15e6e32a0ad4cf0c5d92ba1fdbd9f906
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:33:10.072Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all
---

At the end of September 2026, PK Softech AG was attacked, a software vendor whose pension solution is used by several Swiss pension funds. Data was exfiltrated through this route. According to reporting, those affected include the federal pension fund Publica, Migros Pension Fund, and the City of Zurich Pension Fund, as well as other institutions. Depending on the fund, the data includes names, AHV numbers, marital status, addresses, and IBANs, as well as salary and pension data.

This exfiltration is serious in its own right. It affects identification and financial data for a large share of the working population and, through Publica, large parts of the federal administration. Data of this volume and quality is not merely a problem for individual people. It enables identity theft, targeted deception, and extortion on a large scale, making it a threat to public order and national security. The pension assets themselves were unaffected, but the information platform Cybercrimepolice.ch had already warned of scam calls using the stolen data.

There is also a finding that is difficult to explain for a service provider in this category: **PK Softech AG still has no DMARC record.** Its domain has a strict SPF record, but no DMARC policy (checked on October 18, 2026). This makes it possible to spoof the sender address of the attacked service provider without a receiving mail platform reliably rejecting it. After a data exfiltration, this is the worst possible starting point. Anyone who knows an insured person's name, address, and bank details and can also spoof an email in the name of the service provider or fund can write a message that is almost indistinguishable from a genuine one—for example, asking them to update their IBAN for the next pension payment. DMARC protects precisely against this type of spoofing.

PK Softech is not an isolated case, but the clearest example of a pattern. I reviewed how well Swiss pension institutions and their typical service providers are positioned when it comes to email authentication.

## How the review was conducted

The assessment covers 66 second-pillar pension institutions, ranging from large public-law funds and insurers' collective foundations to smaller joint institutions. Only public DNS records were reviewed: SPF, DKIM, DMARC, MX records, as well as DNSSEC and DANE (TLSA records for mail servers, checked on October 19, 2026). These are the same records that every receiving mail system queries for every incoming message.

The review was entirely passive. Only public DNS was queried; no connections were made to the institutions' systems, no login attempts were made, and no personal data was collected. The assessment reflects only what is publicly available for anyone to see. Corporate collective foundations are grouped under the sponsor's primary domain; individual affiliated foundations may differ.

## The results

| Feature | Share of the 66 institutions |
|---|---|
| SPF present | 88% |
| Strict SPF (`-all`) | 59% |
| DMARC present | 59% |
| DMARC `p=reject` (full protection) | 20% |
| DMARC `p=quarantine` | 26% |
| DMARC `p=none` (monitoring only) | 14% |
| No DMARC | 41% |
| DKIM detectable | 55% |
| DNSSEC | 38% |
| DANE | 9% |

Three points stand out.

**Only 20 percent are effectively protected by DMARC `p=reject`.** Only this policy instructs a receiving mail platform to actually reject spoofed messages. For all others, a spoofed email may end up in the insured person's inbox.

**41 percent have no DMARC.** Their sender domain can readily be used for fraudulent emails, and the institution will not even know about it because DMARC reporting is also absent without DMARC. Notably, this includes funds that run their mail on Microsoft 365, where DMARC can be set up with little effort.

**DMARC `p=none` is not protection.** The 14 percent in this group have set a DMARC policy but left it at the monitoring level. This provides reports but does not reject a single spoofed message. In practice, it is an interim step that often remains in place for months or years.

A similar picture emerges for transport encryption: only 6 of the 66 institutions (9 percent) protect their mail servers with DANE. This allows a sending mail server to use a DNSSEC-signed record to verify that it is securely connected to the correct recipient. DNSSEC is required, and 38 percent of the institutions have already enabled it. For them, DANE is the logical next step.

## Browse all records

The following infographic shows the reviewed status of all 66 pension institutions and 30 service providers. Filters can be used to narrow results by DMARC status, category, and mail provider, and each entry can be expanded to view the full DNS findings.

<div data-dmarc-infographic></div>

## For comparison: Switzerland's largest companies

The [Swiss Mail Security Index](/open-data/mail-security-index) shows how pension institutions compare with the rest of the economy. Using the same passive method, it reviews more than 1,000 of Switzerland's largest companies daily, along with the federal government, cantons, and municipalities, with an interactive map by canton, comparisons by industry and mail provider, and a details page for each organization. All pension institutions and service providers reviewed here are included with their own profile, grouped in the [Pension Institutions](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) industry and updated daily. In the infographic above, the link in the expanded entry leads directly to the respective profile.

The gap is clear: of the 518 companies in the index, 38 percent used DMARC with `p=reject` on October 18, 2026, compared with 20 percent of pension institutions. Five percent of companies have no DMARC at all, compared with 41 percent of pension institutions.

## Suppliers are the bigger issue

The incident at the end of September did not affect a fund, but a shared service provider. I therefore reviewed 30 typical suppliers using the same passive method: vendors of administration software, firms providing outsourced management, actuarial and consulting services, custodian banks, and IT service providers.

On average, service providers are somewhat better positioned than the funds: 40 percent use DMARC with `p=reject`, compared with 20 percent of the institutions. However, the distribution across categories is what matters.

| Category | with DMARC `p=reject` |
|---|---|
| Actuarial and consulting services | good |
| Custodian banks | good |
| IT service providers | mixed |
| Outsourced management | weak |
| Administration software | weakest |

Vendors of administration software are the weakest group. None of the reviewed software providers uses DMARC with full enforcement, and the attacked provider PK Softech has no DMARC record at all. This is precisely the category that processes pension data and is deeply connected to the funds through interfaces, and it is the category involved in the current incident.

This leads to the key point regarding due diligence: A fund with `p=reject` is well protected as long as the attacker wants to spoof its domain. But if they can instead spoof the domain of a poorly protected service provider that already communicates with insured persons, the email is just as credible. Securing an organization's own domain is therefore only the first step. The second is to require its most important service providers to meet the same standard, ideally by contract.

## What this means for the board of trustees

Responsibility for information security rests with the highest governing body of the pension institution. A data exfiltration also triggers specific obligations under the revised Data Protection Act: a breach of data security must be reported to the FDPIC if it is likely to result in a high risk for the affected persons, and those affected must be informed where necessary for their protection.

In this situation, email authentication is the most effective individual measure requiring the least effort. SPF, DKIM, and DMARC require no license and no new software. They consist of DNS records and proper configuration of the existing mail platform. Protection takes effect immediately for every insured person, without requiring them to do anything.

## What needs to be done

For institutions without effective protection, a clear path leads to the goal:

1. **Review SPF and set it strictly.** The SPF record lists authorized sending systems and ends with `-all`, so all other senders are considered unauthorized.
2. **Enable DKIM.** Every outgoing message is cryptographically signed. In Microsoft 365 and common mail gateways, this is a setting, not a project.
3. **Implement DMARC and gradually enforce it.** Start with `p=none` and active reporting to identify legitimate mail traffic. Then use `p=quarantine`, and finally `p=reject`. Depending on the environment, the entire process takes a few weeks, not months.
4. **Review reporting.** DMARC reports show who is sending mail on behalf of the domain and reveal both forgotten legitimate systems and abuse.
5. **At least assess DANE and review mail encryption.** DANE secures transport encryption between mail servers through DNSSEC. Whether the organization's own mail platform and DNS provider support it can be clarified with little effort. For messages containing salary and pension data, there should also be an established process for sending them with end-to-end encryption, for example through a mail gateway with S/MIME or secure delivery through a web portal.
6. **Hold service providers accountable.** Software providers, outsourced management firms, and IT partners should demonstrate the same level of SPF, DKIM, and DMARC. This belongs in contracts and regular supplier reviews.

## Conclusion

The recent attacks targeted a service provider, not the funds themselves. But the consequential damage occurs wherever a sender domain is unprotected—at the fund as well as among its suppliers. Four out of five reviewed pension institutions are not fully protected against spoofed senders, two out of five have no DMARC at all, and vendors of administration software in particular perform worst. The gap is known, inexpensive to close, and takes effect immediately. After a data exfiltration involving names, addresses, and bank details, it is not a minor technical issue but part of the due diligence duty of the highest governing body, throughout the entire supply chain.

## Sources

1.  [SRF: Federal pension fund—how could sensitive data end up at an external software company?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Background on the incident, the types of data affected, and the role of the software supplier.

2.  [cash.ch (AWP): Attack on Publica supplier, salary data may have been affected](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifies PK Softech AG to AWP as the attacked company.

3.  [Netzwoche: Federal pension fund Publica reports data exfiltration](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Report on the attack on the software supplier and the data exfiltration.

4.  [Mimikama: Swiss police warn of possible investment calls after data exfiltration](https://www.mimikama.org/?p=436628): Context for the warning about scam calls using the exfiltrated data.

5.  [FDPIC: DataBreach](https://www.edoeb.admin.ch/de/databreach): Reporting obligation under Art. 24 DPA, reporting portal, and guidance on handling data security breaches.
