---
title: "The pension fund service provider that was attacked has no DMARC record at all"
navTitle: "DMARC pension funds"
description: "PK Softech, the pension fund service provider attacked at the end of September 2026, still has no DMARC record. A passive review of 66 pension institutions and 30 service providers shows that this is not an isolated case: the vendors of administrative software in particular are unprotected."
date: "2026-10-18"
kategorie: "SMTP and mail flow"
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
translationSourceHash: 57506056bfd8dcc96818df43dc71d22a158978bcd7432c91c56eb92698e52fe1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:40:49.368Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all
---

At the end of September 2026, PK Softech AG was attacked, a software vendor whose pension solution is used by several Swiss pension funds. Data was exfiltrated through this route. According to reporting, those affected include the Federal Pension Fund Publica, Migros Pension Fund, and the City of Zurich Pension Fund, as well as other institutions. Depending on the fund, the data includes names, AHV numbers, marital status, addresses, and IBANs, as well as salary and pension data.

This exfiltration is serious in itself. It affects identification and financial data for a large share of the working population and, through Publica, large parts of the federal administration. Data of this volume and quality is not merely a problem for individuals. It enables identity abuse, targeted deception, and extortion on a large scale, making it a threat to public order and national security. The pension assets themselves remained unaffected, but the information platform Cybercrimepolice.ch had already warned about scam calls using the stolen data.

There is also a finding that is difficult to explain for a service provider in this category: **PK Softech AG still has no DMARC record.** Its domain has a strict SPF record but no DMARC policy (checked on October 18, 2026). This means the sender address of the attacked service provider can be spoofed without a receiving mail platform reliably rejecting it. After a data exfiltration, this is the worst possible starting point. Anyone who knows an insured person's name, address, and bank details and can also forge an email in the name of the service provider or fund can write a message that is virtually indistinguishable from a genuine one—for example, a request to update the IBAN for the next pension payment. DMARC protects against precisely this kind of forgery.

PK Softech is not an isolated case, but the clearest example of a pattern. I examined how well Swiss pension institutions and their typical service providers are positioned for email authentication.

## How it was reviewed

The analysis covers 66 second-pillar pension institutions, from large public-law funds and insurers' collective foundations to smaller joint institutions. Only public DNS records were reviewed: SPF, DKIM, DMARC, MX records, as well as DNSSEC and DANE (TLSA records for mail servers, checked on October 19, 2026). These are the same details every receiving mail system queries for every incoming message.

The review was entirely passive. Only public DNS was queried; no connection was made to the institutions' systems, no login was attempted, and no personal data was collected. The analysis reflects only what anyone can already see from public sources. Corporate collective foundations are consolidated under the sponsoring entity's primary domain; individual affiliated foundations may differ.

## The results

| Feature | Share of 66 institutions |
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

**Only 20 percent are effectively protected with DMARC `p=reject`.** Only this policy instructs a receiving mail platform to actually reject spoofed messages. For all others, a forged message may end up in the insured person's inbox.

**41 percent have no DMARC.** Their sender domain can readily be used for fraudulent emails, and the institution will not even learn about it because DMARC's reporting function is also absent without DMARC. Notably, this includes funds that operate their email on Microsoft 365, where DMARC can be configured with little effort.

**DMARC `p=none` is not protection.** The 14 percent in this group have set a DMARC policy but left it at the monitoring stage. It provides reports but rejects not a single spoofed message. In practice, this is an intermediate step that often remains in place for months or years.

A similar picture emerges for transport encryption: only 6 of the 66 institutions (9 percent) protect their mail servers with DANE. This allows a sending mail server to verify, using a DNSSEC-signed record, that it is connected securely to the correct recipient. DNSSEC is a prerequisite, and 38 percent of institutions have already enabled it. For them, DANE is the logical next step.

## All data sets, ready to explore

The following infographic shows the reviewed status of all 66 pension institutions and 30 service providers. The filters allow you to narrow results by DMARC status, category, and mail provider, while every entry can be expanded to view the complete DNS findings.

<div data-dmarc-infographic></div>

## For comparison: Switzerland's largest companies

The [Swiss Email Security Index](/open-data/mail-security-index) shows how pension institutions compare with the rest of the economy. Using the same passive method, it reviews more than 1,000 of Switzerland's largest companies, as well as the federal government, cantons, and municipalities, every day. It includes an interactive map by canton, comparisons by industry and mail provider, and a detail page for each organization. All pension institutions and service providers reviewed here are included with their own profile, grouped under the [Pension institutions](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) industry and updated daily. In the infographic above, the link in the expanded entry leads directly to the relevant profile.

The gap is clear: of the 518 companies in the index, 38 percent used DMARC with `p=reject` on October 18, 2026, compared with 20 percent of pension institutions. Five percent of companies have no DMARC at all, compared with 41 percent of pension institutions.

## Suppliers are the bigger issue

The incident at the end of September did not affect a fund, but a shared service provider. I therefore reviewed 30 typical suppliers using the same passive method: vendors of administrative software, companies providing outsourced management, actuarial services and consulting, custodian banks, and IT service providers.

On average, service providers are somewhat better positioned than the funds: 40 percent use DMARC with `p=reject`, compared with 20 percent of institutions. What matters, however, is the distribution across categories.

| Category | with DMARC `p=reject` |
|---|---|
| Actuarial services and consulting | good |
| Custodian banks | good |
| IT service providers | mixed |
| Outsourced management | weak |
| Administrative software | weakest |

Vendors of administrative software are the weakest group. None of the reviewed software providers uses DMARC with full effect, and the attacked provider PK Softech has no DMARC record at all. This is precisely the category that processes pension data and is deeply connected to the funds through interfaces, and it is the category involved in the current incident.

This leads to the key point for due diligence: A fund with `p=reject` is well protected as long as an attacker wants to spoof its domain. But if the attacker can instead spoof the domain of a poorly protected service provider that already communicates with insured persons, the email is just as credible. Securing an organization's own domain is therefore only the first step. The second is requiring key service providers to meet the same standard, ideally by contract.

## What this means for the board of trustees

Responsibility for information security lies with the pension institution's highest governing body. A data exfiltration also triggers specific obligations under the revised Data Protection Act: a data security breach must be reported to the FDPIC if it is likely to result in a high risk for affected individuals, and those affected must be informed where necessary for their protection.

In this situation, email authentication is the most effective individual measure with the least effort. SPF, DKIM, and DMARC require no license and no new software. They consist of DNS records and a clean configuration of the existing mail platform. Protection takes effect immediately for every insured person, without requiring them to do anything themselves.

## What needs to be done

For institutions without effective protection, a clear path leads to the goal:

1. **Review SPF and set it strictly.** The SPF record lists authorized sending systems and ends with `-all`, so all other senders are considered unauthorized.
2. **Enable DKIM.** Every outgoing message is cryptographically signed. In Microsoft 365 and common mail gateways, this is a setting, not a project.
3. **Introduce DMARC and gradually enforce it.** Start with `p=none` and active reporting to capture legitimate mail traffic. Then use `p=quarantine`, and finally `p=reject`. Depending on the environment, the entire process takes a few weeks, not months.
4. **Review reporting.** DMARC reports show who is sending mail in the domain's name and uncover both overlooked legitimate systems and abuse.
5. **At least assess DANE and review mail encryption.** DANE secures transport encryption between mail servers through DNSSEC. Whether the mail platform and DNS provider support it can be clarified with little effort. For messages containing salary and pension data, there should also be rules for sending them end-to-end encrypted, for example via a mail gateway with S/MIME or secure delivery through a web portal.
6. **Hold service providers accountable.** Software providers, outsourced management, and IT partners should demonstrate that they meet the same standards for SPF, DKIM, and DMARC. This belongs in contracts and regular supplier reviews.

## Conclusion

The recent attacks targeted a service provider, not the funds themselves. But the consequential damage occurs wherever a sender domain is unprotected, whether at the fund or its suppliers. Four out of five reviewed pension institutions are not fully protected against spoofed senders, two out of five have no DMARC at all, and vendors of administrative software in particular perform the worst. The gap is known, inexpensive to close, and effective immediately. After a data exfiltration involving names, addresses, and bank details, it is not a technical side issue but part of the highest governing body's duty of care, across the entire supply chain.

## Sources

1.  [SRF: Federal Pension Fund Publica—how could sensitive data end up at an external software company?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Background on the incident, the types of data affected, and the role of the software supplier.

2.  [cash.ch (AWP): Attack on Publica supplier; salary data may have been affected](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifies PK Softech AG to AWP as the company that was attacked.

3.  [Netzwoche: Federal Pension Fund Publica reports data exfiltration](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Report on the attack on the software supplier and the data exfiltration.

4.  [Mimikama: Swiss police warn of possible investment calls after data exfiltration](https://www.mimikama.org/?p=436628): Context for the warning about scam calls using the exfiltrated data.

5.  [FDPIC: Reporting data security breaches](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Reporting obligation under the revised Data Protection Act and procedures for data security breaches.
