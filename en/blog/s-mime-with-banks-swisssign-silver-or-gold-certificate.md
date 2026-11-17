---
title: "S/MIME for Banks: Legal Framework in Switzerland and the Choice Between Silver and Gold"
navTitle: "S/MIME Silver/Gold"
description: "Which legal requirements apply when a Swiss bank signs and encrypts emails to customers: banking secrecy, FINMA Circular 2023/1, the Data Protection Act, ZertES, rules of evidence, and retention. Also, which SwissSign certificate level follows from these requirements."
date: "2026-11-16"
kategorie: "Email Encryption"
timeToRead: "12 min read"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-with-banks-swisssign-silver-or-gold-certificate"
translationId: "article-5c2fd8d42f088dcb"
translationOf: smime-silver-gold-schweiz
translationSourceHash: f4b52127b6883bb31850668cc18a27ebba43f75200bfdc605d199f427d602ddf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-17T08:47:58.645Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/s-mime-with-banks-swisssign-silver-or-gold-certificate
---

# S/MIME for Banks: Legal Framework in Switzerland and the Choice Between Silver and Gold

In brief: No Swiss law requires a bank to use S/MIME or a specific certificate level. However, banking secrecy, FINMA Circular 2023/1, and the Data Protection Act require customer data to be adequately protected when sent, and the bank must be able to justify and document its solution. An S/MIME signature does not replace a signature, not even with a Gold certificate. As evidence and proof of origin, Gold is clearly superior to the Silver certificate because only Gold contains the verified name of the bank.

The question of which SwissSign certificate a bank needs for sending email comes up regularly in my projects with financial institutions. Behind the technical question is usually a legal one: What must the bank do to fulfill its obligations when sending customer data by email, and what effect does a signed email have?

**Switzerland focus:** This article covers the legal framework in Switzerland. Where EU rules differ, this is indicated in a marked note. It is not a substitute for legal advice in individual cases.

## Overview of the legal framework

| Regulation | Content | Significance for email delivery |
|---|---|---|
| Banking Act, Art. 47 | Banking secrecy, punishable even if negligent | Customer data must not be disclosed without authorization |
| FINMA Circular 2023/1, paras. 73 to 82 | Protection of critical data throughout its entire life cycle | Protection also during delivery, technology-neutral |
| DPA, Arts. 8 and 24; DPO, Arts. 1 to 6 | Data security, notification of breaches | Appropriate measures, notification to the FDPIC in case of high risk |
| ZertES and CO Art. 14(2bis) | Types of electronic signatures, formal validity | S/MIME signatures do not replace a signature |
| CPC Arts. 157 and 177 | Evidence and free assessment of evidence | Signed emails are documents whose evidentiary value is freely assessed |
| CO Art. 958f, GeBüV | Retention of business records | Encrypted emails must remain readable |

None of these enactments mentions S/MIME or a certificate level. They establish protection objectives and legal consequences; the bank decides how to achieve them.

## Banking secrecy

Art. 47 of the Banking Act penalizes anyone who, as an officer, employee, or agent of a bank, discloses a secret entrusted to them in that capacity. Negligent conduct is also punishable. The provision is directed at natural persons, but it is based on the contractual relationship between the customer, as holder of the secret, and the bank.

For email delivery, this means that an unencrypted email containing customer data that is sent to the wrong address or read in transit may constitute negligent disclosure. The bank must therefore ensure, through organizational and technical measures, that employees send customer data only in protected form.

As the holder of the secret, the customer may consent to disclosure. Many banks therefore have customers confirm in their general terms and conditions or a separate agreement that they know and wish to accept the risks of unencrypted email communication. Such consent covers communication only with that customer and only to the agreed extent. It does not replace encryption for documents concerning third parties or for communication with business partners.

## FINMA Circular 2023/1: critical data

FINMA Circular 2023/1 “Operational Risks and Resilience: Banks” has been in force since January 1, 2024, replacing Circular 2008/21. FINMA expanded the previous focus on the confidentiality of customer identification data (CID) to critical data in general. Each bank determines on a risk-based basis what is critical; customer data will generally be included in that category with respect to confidentiality.

The following margin numbers are particularly relevant for email delivery:

| Para. | Content | Relevance to email delivery |
|---|---|---|
| 73 | Identify critical data and categorize it by criticality | Determine which data may only be sent in protected form |
| 74, 75 | Ensure confidentiality, integrity, and availability throughout the entire life cycle | Delivery is part of the life cycle |
| 76 | Protect critical data against access by unauthorized persons | Protect email in transit and at the recipient |
| 79 | Limit increased risks where data is stored abroad or accessed from abroad | Applies to gateways and email platforms located abroad |
| 81 | Report material incidents to FINMA without delay | Such as bulk delivery of customer data to incorrect recipients |
| 82 | Carefully select and monitor service providers processing critical data | Applies to providers of encryption gateways and certification authorities with Managed PKI |

The circular is technology-neutral and mentions neither S/MIME nor certificate levels. In its consultation explanations, FINMA cites data encryption as an example of the special measures used to secure critical data. The bank must define its solution in its own policies and state there which certificate level it uses and why.

## Data protection: DPA and DPO

Art. 8 DPA requires controllers to ensure data security appropriate to the risk. The Data Protection Ordinance specifies this through protection objectives such as confidentiality, integrity, and traceability, but does not prescribe a particular method. S/MIME, a gateway delivering through a web portal, or an exchange platform all meet the requirement if used in a manner appropriate to the risk.

If a data security breach is likely to result in a high risk to the personality or fundamental rights of the individuals concerned, the bank must report it to the FDPIC as soon as possible under Art. 24 DPA. This reporting obligation exists alongside the obligation to report to FINMA. If the email was encrypted, the risk in the event of misdelivery is generally low and no report is required.

## Legal effect of an S/MIME signature

### Not a signature within the meaning of the CO

ZertES distinguishes between electronic, advanced, regulated, and qualified electronic signatures, as well as the regulated electronic seal for companies. Under Art. 14(2bis) CO, only a qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature.

SwissSign S/MIME certificates are issued under the CA/Browser Forum rules and the ETSI LCP (Silver) and NCP (Gold) policies. They are not qualified certificates under ZertES. A bank therefore cannot make declarations validly in the required form by S/MIME-signed email where legislation or a contract requires written form, even with Gold.

### Personal signature and gateway signature

An advanced signature requires, among other things, that the signature key be under the sole control of the signing person. If a gateway centrally signs all outgoing emails with keys held on the gateway, this requirement is not met. The signature then proves that the email originated from the bank’s system and was not altered, but not that a specific person approved it. In substance, it is therefore more like an organization’s seal, without meeting the requirements for a regulated seal under ZertES.

### Evidentiary value

Under Art. 177 CPC, an email is a document, and the court freely assesses its evidentiary value under Art. 157 CPC. A valid S/MIME signature shows that the email has not been altered since signing and was signed with the key of a particular certificate. How much that says about the sender depends on what the certification authority verified before issuance. This is where the legally relevant difference between Silver and Gold lies.

## Silver or Gold from a legal perspective

Since September 1, 2023, publicly trusted S/MIME certificates have been subject to the `S/MIME Baseline Requirements` of the CA/Browser Forum. SwissSign assigns its products to these validation types in its certificate profiles (CPR S/MIME):

| Product | CA/B Forum type | What SwissSign verifies | What the certificate contains | What the signature states |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control of the mailbox | Only the email address | Someone with access to this mailbox |
| E-Mail ID Gold | `Sponsor-validated` | Organization and person | Name, organization, organization identifier, country, email address | A verified person at this bank |
| Organization Validated | `Organization-validated` | Organization | Name of the organization, no person | A mailbox of this bank |

The levels are cryptographically equivalent. For encryption, and thus for the protection obligations arising from banking secrecy, the FINMA circular, and the DPA, the level is therefore irrelevant.

For the signature, the difference is significant. Anyone who can receive emails at an address can obtain a Silver certificate. Someone who registers a similarly named domain, for example `musterbank-ch.com` instead of `musterbank.ch`, can use it to send emails that are signed just as validly as the bank’s genuine emails. In a dispute over whether an email originated from the bank, a Silver signature therefore proves little. By contrast, a Gold or Organization Validated certificate contains the name of the bank verified by SwissSign. Gold additionally associates it with a person whose identity the bank confirms as the registration authority within the Managed PKI.

## Retention: CO and GeBüV

Under Art. 958f CO, business books and accounting records must be retained for ten years, and the Ordinance on Business Records (GeBüV) requires electronically retained documents to remain readable throughout that period. Banks also have supervisory and contractual retention obligations. This results in two requirements for S/MIME operations:

- Encrypted emails must be capable of being decrypted throughout the entire retention period. The bank must therefore securely retain the private keys of expired certificates or decrypt the emails before archiving.
- If a signature is to serve as evidence years later, it must remain possible to determine that the certificate was valid when the email was signed. The validation results or revocation information at the time of signing must therefore also be archived.

**EU note:** In the EU, Art. 25(2) eIDAS likewise provides that only a qualified electronic signature is equivalent to a handwritten signature. Since January 17, 2025, the DORA Regulation has also applied to financial undertakings in the EU, with its own requirements for ICT risk management.

## Example: One gateway for all employees

Many banks use a central encryption gateway, such as PrivaSphere, SEPPmail, or TotemoMail, so that all employees can encrypt and sign emails without having to manage certificates themselves. The keys are held on the gateway, which signs and encrypts on behalf of the individual mailboxes. Based on the preceding sections, the following choice applies to this setup:

| Mailbox | Level | Reason |
|---|---|---|
| Personal mailboxes | E-Mail ID Gold | Identifies the bank as the sender with a verified name and organization identifier |
| Functional mailboxes (customer service, payment transactions) | Organization Validated | Identifies the bank without naming an individual person |
| None | E-Mail ID Silver | Proves only access to a mailbox; emails from similarly named domains are signed just as validly |

Because the gateway signs rather than the individual person, the keys are not under the sole control of employees. Legally, the signature functions as a seal of the bank rather than a personal signature. It is therefore crucial that the organization is identified in the certificate as verified, which only Gold and Organization Validated provide. The level is irrelevant to encryption itself.

Before implementation, the following points should be clarified with the gateway provider and certification authority:

- **Issuance and renewal:** Whether the gateway obtains and renews Gold certificates automatically through the certification authority’s Managed PKI. For Gold, the bank, as the registration authority, confirms the identity of each person; this requires a process for hires and departures, name changes, and revocations.
- **Service provider assessment:** Gateway providers and Managed PKI process critical data within the meaning of para. 82 of FINMA Circular 2023/1. The location of keys and data must be documented; where data is stored abroad or accessed from abroad, para. 79 also applies.
- **Archiving:** The private keys of expired certificates and the validation data at the time of signing must remain available throughout the entire retention period.
- **Policy:** The selected levels and approved procedures belong in the internal policy, which serves as evidence for FINMA and audit purposes.

## Recommendation

- **Define a policy:** The bank defines in an internal policy which data may only be sent encrypted, which methods are approved for this purpose, and which certificate levels it uses. This is the basis for demonstrating compliance with the requirements of the FINMA circular and the DPA.
- **Gold for personal mailboxes, Organization Validated for functional mailboxes:** Only these levels identify the bank in the certificate as the sender and therefore have evidentiary value beyond access to a mailbox. Silver is not suitable for communication with customers.
- **Clarify consents:** Anyone communicating with customers by unencrypted email needs their documented consent. It does not replace encryption for third parties and business partners.
- **Do not use email for declarations subject to formal requirements:** Declarations requiring written form need a qualified electronic signature or a handwritten signature.
- **Assess service providers:** Gateway providers and the certification authority’s Managed PKI fall under para. 82 of the FINMA circular and, where located abroad, also under para. 79.
- **Archive keys and validation data:** Private keys for decryption and data for signature validation must remain available throughout the retention period.
- **Combine reporting processes:** Misdelivery of customer data can trigger a report to FINMA (para. 81) and to the FDPIC (Art. 24 DPA) at the same time. Both obligations can be covered in a joint incident process.

## Sources

1.  [FINMA Circular 2023/1 “Operational Risks and Resilience: Banks”](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): paras. 73 to 82 on critical data, in force since January 1, 2024.

2.  [datenrecht.ch: FINMA Circular 2023/1 with explanations](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): revised text with FINMA’s consultation comments, including data encryption as a protective measure.

3.  [Banking Act (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 on banking secrecy.

4.  [Swissbanking: Banking secrecy and cloud, 2019 legal opinion](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): legal opinion on the interpretation of Art. 47 BankG, including customer consent.

5.  [Federal Act on Data Protection (DPA)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 on data security, Art. 24 on notification of data security breaches.

6.  [Data Protection Ordinance (DPO)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Arts. 1 to 6 on the principles and protection objectives of data security.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types of electronic signatures and seals, requirements for qualified certificates.

8.  [Swiss Code of Obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14(2bis) on the equivalence of the qualified electronic signature to a handwritten signature, Art. 958f on retention.

9.  [Swiss Civil Procedure Code (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 on free assessment of evidence, Art. 177 on the concept of documents.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSign certificate profiles (as of September 18, 2025), assignment of Silver, Gold, and Organization Validated to validation types and Subject fields.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): binding rules for publicly trusted S/MIME certificates, definition of validation types.
