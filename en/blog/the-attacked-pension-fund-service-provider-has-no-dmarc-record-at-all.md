---
title: "The attacked pension fund service provider has no DMARC record at all"
navTitle: "Pension Fund DMARC"
description: "PK Softech, the pension fund service provider attacked at the end of September 2026, still has no DMARC record. A passive review of 66 pension institutions and 30 service providers shows that this is not an isolated case: vendors of administration software in particular are unprotected."
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:14:27.410Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all
---

At the end of September 2026, PK Softech AG was attacked, a software vendor whose pension solution is used by several Swiss pension funds. Data was exfiltrated through this route. According to reporting, those affected include the Publica federal pension fund, Migros' pension fund, and the City of Zurich pension fund, as well as other institutions. Depending on the fund, this involves names, AHV numbers, marital status, addresses, and IBANs, as well as salary and pension data.

This exfiltration is serious in itself. It affects identification and financial data for a large part of the working population and, through Publica, large parts of the federal administration. Data of this quantity and quality is not just a problem for individual people. It enables identity theft, targeted deception, and extortion on a large scale, and thus poses a threat to public order and national security. The pension assets themselves remained unaffected, but the Cybercrimepolice.ch information platform had already warned of fraudulent calls using the stolen data.

There is also a finding that is difficult to explain for a service provider in this category: **PK Softech AG still has no DMARC record.** Its domain has a strict SPF record, but no DMARC policy (checked on October 18, 2026). This makes it possible to spoof the attacked service provider's sender address without a receiving mail platform reliably rejecting it. After a data exfiltration, this is the worst possible starting point. Anyone who knows an insured person's name, address, and bank account details and can also spoof an email in the name of the service provider or fund can write a message that is almost indistinguishable from a real one, such as a request to update the IBAN for the next pension payment. DMARC protects precisely against this kind of spoofing.

PK Softech is not an isolated case, but rather the clearest example of a pattern. I examined how well Swiss pension institutions and their typical service providers are positioned when it comes to email authentication.

## How the review was conducted

The analysis covers 66 second-pillar pension institutions, from large public-law funds and insurers' collective foundations to smaller joint institutions. Only public DNS records were examined: SPF, DKIM, DMARC, MX records, as well as DNSSEC and DANE (TLSA records of the mail servers, checked on October 19, 2026). These are the same records that every receiving mail system queries for every incoming message.

The review was entirely passive. Only public DNS was queried; no connection was established to the institutions' systems, no login was attempted, and no personal data was collected. The analysis only reflects what is publicly available for anyone to see anyway. Corporate collective foundations are consolidated under the sponsor's primary domain; individual affiliated foundations may differ.

## The result

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

**Only 20 percent are effectively protected with DMARC `p=reject`.** Only this policy instructs a receiving mail platform to actually reject spoofed messages. For everyone else, a spoofed message will likely end up in the insured person's inbox.

**41 percent have no DMARC.** Their sender domain can readily be used for fraudulent emails, and the institution will not even know about it, because without DMARC, the reporting function is also absent. Notably, this includes funds that run their email on Microsoft 365, where DMARC can be configured with little effort.

**DMARC `p=none` is not protection.** The 14 percent in this group have set a DMARC policy but left it at the monitoring stage. This provides reports but does not reject a single spoofed message. In practice, it is an intermediate step that often remains in place for months or years.

A similar picture emerges for transport encryption: only 6 of the 66 institutions (9 percent) secure their mail servers with DANE. With it, a sending mail server uses a DNSSEC-signed record to verify that it is encryptedly connected to the correct recipient. DNSSEC is a prerequisite, and 38 percent of the institutions have already enabled it. For them, DANE is the obvious next step.

## All records, clickable

The following infographic shows the reviewed status of all 66 pension institutions and 30 service providers. The filters can be used to narrow results by DMARC status, category, and mail provider, and every entry can be expanded to display the full DNS findings.

<div data-dmarc-infographic></div>

## For comparison: Switzerland's largest companies

How the pension institutions compare with the rest of the economy is shown by the [Swiss Mail Security Index](/open-data/mail-security-index). Using the same passive method, it examines more than 1,000 of Switzerland's largest companies every day, along with the federal government, cantons, and municipalities, with an interactive map by canton, comparisons by industry and mail provider, and a detail page for each organization. All pension institutions and service providers reviewed here are included there with their own profiles, grouped in the [Pension Institutions](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) industry and updated daily. In the infographic above, the link in the expanded entry leads directly to the relevant profile.

The gap is clear: Of the 518 companies in the index, 38 percent used DMARC with `p=reject` on October 18, 2026, compared with 20 percent of pension institutions. Five percent of companies have no DMARC at all, compared with 41 percent of pension institutions.

## Suppliers are the bigger issue

The incident at the end of September affected not a fund, but a shared service provider. That is why I examined 30 typical suppliers using the same passive method: vendors of administration software, companies providing outsourced management, actuarial services and consulting, custodian banks, and IT service providers.

On average, service providers fare somewhat better than the funds: 40 percent use DMARC with `p=reject`, compared with 20 percent of the institutions. The distribution across categories is decisive, however.

| Category | with DMARC `p=reject` |
|---|---|
| Actuarial services and consulting | good |
| Custodian banks | good |
| IT service providers | mixed |
| Outsourced management | weak |
| Administration software | weakest |

Vendors of administration software are the weakest group. None of the reviewed software providers use DMARC with full effect, and the attacked provider PK Softech has no DMARC record at all. This is precisely the category that processes pension data and is deeply connected to the funds through interfaces, and it is the category involved in the current incident.

This leads to the key point for due diligence: A fund with `p=reject` is well protected as long as the attacker wants to spoof its domain. But if they can instead spoof the domain of a poorly protected service provider that already communicates with insured persons, the email is just as credible. Securing the organization's own domain is therefore only the first step. The second is to require key service providers to meet the same standard, ideally by contract.

## What this means for the board of trustees

Responsibility for information security lies with the highest governing body of the pension institution. A data exfiltration also triggers specific obligations under the revised Data Protection Act: A data security breach must be reported to the FDPIC if it is likely to result in a high risk to the affected persons, and those affected must be informed where necessary for their protection.

In this situation, email authentication is the most effective individual measure requiring the least effort. SPF, DKIM, and DMARC require no license and no new software. They consist of DNS records and a proper configuration of the existing mail platform. Protection takes effect immediately for every insured person, without them having to do anything themselves.

## What needs to be done

For institutions without effective protection, a clear path leads to the goal:

1. **Check SPF and set it strictly.** The SPF record lists authorized sending systems and ends with `-all`, so that all other senders are considered unauthorized.
2. **Enable DKIM.** Every outgoing message is cryptographically signed. In Microsoft 365 and common email gateways, this is a setting, not a project.
3. **Implement DMARC and progressively enforce it.** Start with `p=none` and active reporting to capture legitimate mail traffic. Then use `p=quarantine`, and finally `p=reject`. Depending on the environment, the entire process takes a few weeks, not months.
4. **Evaluate reporting.** DMARC reports show who is sending email on behalf of the domain and reveal both forgotten legitimate systems and abuse.
5. **At least assess DANE and review email encryption.** DANE secures transport encryption between mail servers via DNSSEC. Whether the organization's mail platform and DNS provider support it can be clarified with little effort. For messages containing salary and pension data, it should also be defined how they are sent end-to-end encrypted, for example through an email gateway with S/MIME or secure delivery via a web portal.
6. **Hold service providers accountable.** Software vendors, outsourced management providers, and IT partners should demonstrate the same level of SPF, DKIM, and DMARC. This belongs in contracts and regular supplier reviews.

## Conclusion

The recent attacks targeted a service provider, not the funds themselves. But the resulting harm occurs wherever a sender domain is unprotected, whether at the fund or among its suppliers. Four out of five reviewed pension institutions are not fully protected against spoofed senders, two out of five have no DMARC at all, and vendors of administration software in particular perform the worst. The gap is known, inexpensive to close, and takes effect immediately. After a data exfiltration involving names, addresses, and bank account details, it is not a technical side issue, but part of the duty of care of the highest governing body, across the entire supply chain.

## Change log

- **October 25, 2026:** Updated the source link to the FDPIC's new address.

## Sources

1.  [SRF: Federal pension fund Publica—how could sensitive data end up with an external software company?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Background on the incident, the types of data affected, and the role of the software supplier.

2.  [cash.ch (AWP): Attack on Publica supplier; salary data may have been affected](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifies PK Softech AG to AWP as the company that was attacked.

3.  [Netzwoche: Federal pension fund Publica reports data exfiltration](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Report on the attack on the software supplier and the data exfiltration.

4.  [Mimikama: Swiss police warn of possible investment scam calls after data exfiltration](https://www.mimikama.org/?p=436628): Context for the warning about fraudulent calls using the exfiltrated data.

5.  [FDPIC: DataBreach](https://www.edoeb.admin.ch/de/databreach): Reporting obligation under Article 24 of the Data Protection Act, reporting portal, and guidance on responding to data security breaches.
