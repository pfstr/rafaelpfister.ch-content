---
title: "The attacked pension fund service provider has no DMARC record at all"
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
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/en/blog/the-attacked-pension-fund-service-provider-has-no-dmarc-record-at-all
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:52:41.705Z
translationReview: required
---

At the end of September 2026, PK Softech AG was attacked, a software vendor whose pension solution is used by several Swiss pension funds. Data was exfiltrated through this route. According to reporting, those affected include the Publica federal pension fund, Migros Pension Fund, and the Pension Fund of the City of Zurich, as well as other institutions. Depending on the fund, the data includes names, AHV numbers, marital status, addresses, and IBANs, as well as salary and pension data.

This exfiltration is serious in itself. It affects identification and financial data for a large share of the working population and, through Publica, large parts of the federal administration. Data of this volume and quality is not merely a problem for individual people. It enables identity abuse, targeted deception, and extortion on a large scale, making it a threat to public order and national security. The pension assets themselves remained unaffected, but the cybercrime police had already warned about scam calls using the stolen data.

There is also a finding that is difficult to explain for a service provider in this category: **PK Softech AG still has no DMARC record.** Its domain has a strict SPF record but no DMARC policy (checked on October 18, 2026). This means the sender address of the attacked service provider can be spoofed without a receiving mail platform reliably rejecting it. After a data exfiltration, this is the worst possible starting point. Anyone who knows an insured person's name, address, and bank details and can also spoof an email in the name of the service provider or fund can write a message that is nearly indistinguishable from a genuine one—for example, asking them to update their IBAN for the next pension payment. DMARC protects against precisely this type of spoofing.

PK Softech is not an isolated case, but rather the clearest example of a pattern. I reviewed how well Swiss pension institutions and their typical service providers are positioned for email authentication.

## How the review was conducted

The analysis covers 66 second-pillar pension institutions, from large public-law funds and insurers' collective foundations to smaller collective institutions. Only public DNS records were reviewed: SPF, DKIM, DMARC, MTA-STS, TLS-RPT, and MX records. These are the same records that every receiving mail system queries for every incoming message.

The review was entirely passive. Only public DNS was queried; no connection was made to the institutions' systems, no login was attempted, and no personal data was collected. The analysis reflects only what is publicly visible to anyone anyway. Corporate collective foundations are consolidated under the sponsoring entity's main domain; individual affiliated foundations may differ.

## The results

| Feature | Share of 66 institutions |
|---|---|
| SPF present | 88% |
| Strict SPF (`-all`) | 59% |
| DMARC present | 59% |
| DMARC `p=reject` (full protection) | 20% |
| DMARC `p=quarantine` | 26% |
| DMARC `p=none` (monitoring only) | 14% |
| No DMARC | 41% |
| DKIM detectable | 55% |
| MTA-STS | 2% |
| TLS-RPT | 5% |

Three points stand out.

**Only 20 percent are effectively protected by DMARC `p=reject`.** Only this policy instructs a receiving mail platform to actually reject spoofed messages. For everyone else, a spoofed email will generally end up in the insured person's inbox.

**41 percent have no DMARC.** Their sender domain can readily be used for fraudulent emails, and the institution will not even know about it because without DMARC, the reporting function is also absent. Notably, this includes funds that run their email on Microsoft 365, where DMARC can be set up with little effort.

**DMARC `p=none` is not protection.** The 14 percent in this group have set a DMARC policy but left it at the monitoring level. That provides reports, but does not reject a single spoofed message. In practice, this is an intermediate step that often remains in place for months or years.

There is also a consistent finding: at 2 percent, MTA-STS is practically nonexistent. Without MTA-STS, transport encryption for incoming email is not enforced, which favors an attacker with access to the transport path.

## All records at a glance

The following infographic shows the reviewed status of all 66 pension institutions and 30 service providers. The filters allow narrowing by DMARC status, category, and mail provider, and each entry can be expanded to view the complete DNS findings.

<div data-dmarc-infographic></div>

## For comparison: Switzerland's largest companies

The [Swiss Email Security Index](/open-data/mail-security-index) shows how pension institutions compare with the rest of the economy. Using the same passive method, it reviews more than 500 of Switzerland's largest companies every day, with an interactive map by canton, comparisons by industry and mail provider, and a detailed page for each company. This also includes the banks, insurers, and IT service providers that pension institutions work with.

The gap is clear: Of the 518 companies in the index, 38 percent used DMARC with `p=reject` on October 18, 2026, compared with 20 percent of pension institutions. Five percent of companies have no DMARC at all, compared with 41 percent of pension institutions.

## Suppliers are the bigger issue

The incident at the end of September affected not a fund, but a shared service provider. I therefore reviewed 30 typical suppliers using the same passive method: vendors of administrative software, companies providing outsourced administration, actuarial services and consulting, custodian banks, and IT service providers.

On average, service providers are in a slightly better position than funds: 40 percent use DMARC with `p=reject`, compared with 20 percent of the institutions. However, the distribution across categories is crucial.

| Category | With DMARC `p=reject` |
|---|---|
| Actuarial services and consulting | good |
| Custodian banks | good |
| IT service providers | mixed |
| Outsourced administration | weak |
| Administrative software | weakest |

Vendors of administrative software are the weakest group. None of the reviewed software vendors uses fully effective DMARC, and the attacked vendor PK Softech has no DMARC record at all. This is precisely the category that processes pension data and is deeply connected to funds through interfaces, and it is the category involved in the current incident.

This leads to the real due-diligence point: A fund with `p=reject` is well protected as long as the attacker wants to spoof its domain. But if the attacker can instead spoof the domain of a poorly protected service provider that already communicates with insured persons, the email is just as credible. Securing one's own domain is therefore only the first step. The second is requiring the most important service providers to meet the same standard, ideally through contractual obligations.

## What this means for the board of trustees

Responsibility for information security rests with the pension institution's highest governing body. A data exfiltration also triggers specific obligations under the revised Data Protection Act: A breach of data security must be reported to the FDPIC if it is likely to result in a high risk for the affected individuals, and those affected must be informed where necessary for their protection.

In this situation, email authentication is the most effective individual measure requiring the least effort. SPF, DKIM, and DMARC require neither a license nor new software. They consist of DNS records and a clean configuration of the existing mail platform. The protection takes effect immediately for every insured person, without them having to do anything themselves.

## What needs to be done

For institutions without effective protection, there is a clear path to the goal:

1. **Review SPF and set it strictly.** The SPF record lists authorized sending systems and ends with `-all`, so all other senders are considered unauthorized.
2. **Enable DKIM.** Every outgoing message is cryptographically signed. In Microsoft 365 and common mail gateways, this is a setting, not a project.
3. **Introduce DMARC and tighten it gradually.** Start with `p=none` and active reporting to capture legitimate email traffic. Then use `p=quarantine`, and finally `p=reject`. Depending on the environment, the entire process takes a few weeks, not months.
4. **Review reporting.** DMARC reports show who is sending email in the domain's name and uncover both forgotten legitimate systems and abuse.
5. **Add MTA-STS and TLS-RPT.** Both enforce and monitor transport encryption for incoming email.
6. **Hold service providers accountable.** Software vendors, outsourced administration providers, and IT partners should demonstrate the same standard for SPF, DKIM, and DMARC. This belongs in contracts and regular supplier reviews.

## Conclusion

The recent attacks targeted a service provider, not the funds themselves. But the resulting damage occurs wherever a sender domain is unprotected—at the fund just as much as among its suppliers. Four out of five reviewed pension institutions are not fully protected against spoofed senders, two out of five have no DMARC at all, and vendors of administrative software in particular perform the worst. The gap is known, inexpensive to close, and effective immediately. After a data exfiltration involving names, addresses, and bank details, it is not a technical side issue, but part of the highest governing body's duty of care across the entire supply chain.

## Sources

1.  [SRF: Federal pension fund—how could sensitive data end up at an external software company?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Background on the incident, the types of data affected, and the software supplier's role.

2.  [cash.ch (AWP): Attack on Publica supplier; salary data may have been affected](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifies PK Softech AG to AWP as the attacked company.

3.  [Netzwoche: Publica federal pension fund reports data exfiltration](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Report on the attack against the software supplier and the data exfiltration.

4.  [Mimikama: Swiss police warn of possible investment scam calls after data exfiltration](https://www.mimikama.org/?p=436628): Context for the warning about scam calls using the exfiltrated data.

5.  [FDPIC: Reporting data security breaches](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Reporting obligation under the revised Data Protection Act and procedures for data security breaches.
