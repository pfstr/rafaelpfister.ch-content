---
title: "S/MIME for Banks: Legal Framework in Switzerland and the Choice Between Silver and Gold"
navTitle: "S/MIME Silver/Gold"
description: "Which legal requirements apply when a Swiss bank signs and encrypts emails to customers: banking secrecy, FINMA Circular 2023/1, the Data Protection Act, ZertES, evidentiary law, and retention. Also, which SwissSign certificate level follows from these requirements."
date: "2026-11-16"
kategorie: "Email Encryption"
timeToRead: "11 min read"
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
translationSourceHash: 60de430124ebe20a3001bd48fc55affcb921141039e7c85303da9c7216b01e89
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:19:48.036Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/s-mime-with-banks-swisssign-silver-or-gold-certificate
---

# S/MIME for Banks: Legal Framework in Switzerland and the Choice Between Silver and Gold

In brief: No Swiss law requires a bank to use S/MIME or a particular certificate level. However, banking secrecy, FINMA Circular 2023/1, and the Data Protection Act require customer data to be adequately protected when sent, and the bank must be able to justify and document its solution. An S/MIME signature does not replace a signature, even with a Gold certificate. As evidence and proof of origin, Gold is clearly superior to a Silver certificate because only Gold contains the verified name of the bank.

The question of which SwissSign certificate a bank needs for sending email regularly arises in my projects with financial institutions. Behind the technical question is usually a legal one: What must the bank do to fulfill its obligations when sending customer data by email, and what legal effect does a signed email have?

**Switzerland focus:** This article addresses the legal situation in Switzerland. Where EU rules differ, this is indicated in a labeled note. It does not replace legal advice in individual cases.

## Overview of the legal framework

| Regulation | Content | Significance for sending email |
|---|---|---|
| Banking Act, Art. 47 | Banking secrecy, punishable even if breached negligently | Customer data must not be disclosed without authorization |
| FINMA Circular 2023/1, margin nos. 73 to 82 | Protection of critical data throughout its entire lifecycle | Protection also during transmission, technology-neutral |
| Data Protection Act, Arts. 8 and 24; Data Protection Ordinance, Arts. 1 to 6 | Data security, reporting of breaches | Appropriate measures, report to the FDPIC where there is high risk |
| ZertES and Code of Obligations, Art. 14 para. 2bis | Types of electronic signatures, formal validity | S/MIME signatures do not replace a signature |
| Civil Procedure Code, Arts. 157 and 177 | Evidence and free assessment of evidence | Signed emails are documents whose evidentiary value is freely assessed |
| Code of Obligations, Art. 958f; GeBüV | Retention of business records | Encrypted emails must remain readable |

None of these laws mentions S/MIME or a certificate level. They establish protection objectives and legal consequences; the bank decides how to achieve them.

## Banking secrecy

Art. 47 of the Banking Act punishes anyone who, as an officer, employee, or agent of a bank, discloses a secret entrusted to them in that capacity. Negligent conduct is also punishable. The provision is addressed to natural persons; however, it is based on the contractual relationship between the customer as holder of the secret and the bank.

For sending email, this means: An unencrypted email containing customer data that is sent to the wrong address or read in transit may constitute negligent disclosure. The bank must therefore use organizational and technical measures to ensure that employees send customer data only in protected form.

As the holder of the secret, the customer can consent to disclosure. Many banks therefore obtain confirmation in their general terms and conditions or in a separate agreement that the customer knows and wishes to accept the risks of unencrypted email communication. Such consent covers only communication with that customer and only to the agreed extent. It does not replace encryption for documents concerning third parties or for communication with business partners.

## FINMA Circular 2023/1: critical data

FINMA Circular 2023/1, “Operational Risks and Resilience: Banks,” has been in force since January 1, 2024, and replaced Circular 2008/21. FINMA expanded the previous focus on the confidentiality of customer identification data (CID) to critical data in general. Each bank determines what is critical based on risk; customer data will generally fall into this category with regard to confidentiality.

For sending email, the following margin numbers are particularly relevant:

| Margin no. | Content | Relevance to sending email |
|---|---|---|
| 73 | Identify critical data and categorize it by criticality | Determine which data may be sent only in protected form |
| 74, 75 | Ensure confidentiality, integrity, and availability throughout the entire lifecycle | Sending is part of the lifecycle |
| 76 | Protect critical data against access by unauthorized persons | Protection of email in transit and at the recipient |
| 79 | Limit increased risks associated with storage in or access from abroad | Applies to gateways and email platforms located abroad |
| 81 | Report material incidents to FINMA without delay | For example, mass sending of customer data to incorrect recipients |
| 82 | Carefully select and monitor service providers processing critical data | Applies to providers of encryption gateways and certification authorities with managed PKI |

The circular is technology-neutral and mentions neither S/MIME nor certificate levels. In its consultation explanations, FINMA cites data encryption as an example of the special measures used to secure critical data. The bank must define its solution in its own internal directives and also specify there which certificate level it uses and why.

## Data protection: Data Protection Act and Data Protection Ordinance

Art. 8 of the Data Protection Act requires controllers to ensure data security appropriate to the risk. The Data Protection Ordinance elaborates on this with protection objectives such as confidentiality, integrity, and traceability, but does not prescribe a particular method. S/MIME, a gateway delivering through a web portal, or an exchange platform all meet the requirement if used in a manner appropriate to the risk.

If a data security breach is likely to result in a high risk to the personality or fundamental rights of the persons concerned, the bank must report it to the FDPIC as soon as possible under Art. 24 of the Data Protection Act. This reporting obligation exists alongside the obligation to report to FINMA. If the email was encrypted, the risk in the event of misdelivery is generally low and no report is required.

## The legal effect of an S/MIME signature

### Not a signature within the meaning of the Code of Obligations

ZertES distinguishes between electronic, advanced, regulated, and qualified electronic signatures, as well as regulated electronic seals for companies. Under Art. 14 para. 2bis of the Code of Obligations, only a qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature.

SwissSign S/MIME certificates are issued under the CA/Browser Forum rules and the ETSI policies LCP (Silver) and NCP (Gold), respectively. They are not qualified certificates under ZertES. A bank therefore cannot make declarations that require written form by law or contract in formally valid form through an S/MIME-signed email, not even with Gold.

### Personal signature and gateway signature

Among other things, an advanced signature requires that the signature key remain under the sole control of the signing person. If a gateway centrally signs all outgoing emails using keys stored on the gateway, this requirement is not met. The signature then proves that the email originated from the bank’s system and has not been altered, but not that a particular person approved it. Its substance is therefore closer to an organizational seal, without meeting the requirements for a regulated seal under ZertES.

### Evidentiary value

Under Art. 177 of the Civil Procedure Code, an email is a document, and the court freely assesses its evidentiary value under Art. 157 of the Civil Procedure Code. A valid S/MIME signature shows that the email has not been altered since signing and was signed using the key of a particular certificate. How much this says about the sender depends on what the certification authority verified before issuance. This is where the legally relevant difference between Silver and Gold lies.

## Silver or Gold from a legal perspective

Since September 1, 2023, publicly trusted S/MIME certificates have been subject to the `S/MIME Baseline Requirements` of the CA/Browser Forum. SwissSign assigns its products to these validation types in its certificate profiles (CPR S/MIME):

| Product | CA/B Forum type | What SwissSign verifies | What the certificate contains | What the signature states |
|---|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control over the mailbox | Only the email address | Someone with access to this mailbox |
| E-Mail ID Gold | `Sponsor-validated` | Organization and person | Name, organization, organization identifier, country, email address | A verified person at this bank |
| Organization Validated | `Organization-validated` | Organization | Organization name, no person | A mailbox of this bank |

The levels are cryptographically equivalent. For encryption, and thus for the protection obligations arising from banking secrecy, the FINMA Circular, and the Data Protection Act, the level therefore makes no difference.

For signatures, the difference is substantial. Anyone who can receive email at an address can obtain a Silver certificate. Someone who registers a similarly named domain, such as `musterbank-ch.com` instead of `musterbank.ch`, can use it to send emails with signatures that are just as valid as the bank’s genuine emails. In a dispute over whether an email originated from the bank, a Silver signature therefore proves little. By contrast, a Gold or Organization Validated certificate contains the bank’s name verified by SwissSign. With Gold, there is also an assignment to a person whose identity the bank confirms as the registration authority within the managed PKI.

## Retention: Code of Obligations and GeBüV

Under Art. 958f of the Code of Obligations, business records and accounting vouchers must be retained for ten years, and the Ordinance on the Keeping and Retention of Business Records (GeBüV) requires electronically retained documents to remain readable during that time. Banks are also subject to supervisory and contractual retention obligations. This results in two requirements for S/MIME operations:

- Encrypted emails must be decryptable throughout the entire retention period. The bank must therefore securely retain the private keys of expired certificates or decrypt emails before archiving them.
- If a signature is to serve as evidence years later, it must remain possible to establish that the certificate was valid at the time of signing. The verification results or revocation information applicable at the time of signing must therefore also be archived.

**EU note:** In the EU, Art. 25(2) eIDAS likewise recognizes only the qualified electronic signature as equivalent to a handwritten signature. Since January 17, 2025, financial institutions in the EU have additionally been subject to the DORA Regulation, with its own requirements for ICT risk management.

## Recommendation

- **Establish an internal directive:** The bank defines in an internal directive which data may be sent only in encrypted form, which procedures are permitted for this purpose, and which certificate levels it uses. This is the basis for demonstrating compliance with the requirements of the FINMA Circular and the Data Protection Act.
- **Gold for personal mailboxes, Organization Validated for functional mailboxes:** Only these levels identify the bank as the sender in the certificate and therefore have evidentiary value beyond access to a mailbox. Silver is not suitable for communication with customers.
- **Clarify consent:** Anyone communicating with customers by unencrypted email needs their documented consent. It does not replace encryption in relation to third parties and business partners.
- **Do not use email for declarations requiring a specific form:** Declarations requiring written form need a qualified electronic signature or a handwritten signature.
- **Review service providers:** Gateway providers and the certification authority’s managed PKI fall under margin no. 82 of the FINMA Circular, and, where locations abroad are involved, additionally under margin no. 79.
- **Archive keys and verification data:** Private keys for decryption and data for signature verification must remain available throughout the retention period.
- **Combine reporting processes:** Misdelivery of customer data can simultaneously trigger a report to FINMA (margin no. 81) and to the FDPIC (Art. 24 of the Data Protection Act). Both obligations can be handled in a single incident process.

## Sources

1.  [FINMA Circular 2023/1 “Operational Risks and Resilience: Banks”](https://www.finma.ch/de/~/media/finma/dokumente/dokumentencenter/myfinma/rundschreiben/finma-rs-2023-01-20221207.pdf): Margin nos. 73 to 82 on critical data, in force since January 1, 2024.

2.  [datenrecht.ch: FINMA Circular 2023/1 with explanations](https://datenrecht.ch/wp-content/uploads/FINMA-RS-OpRisk-%E2%80%93-Text-revidiert-1.1.2024.pdf): Revised text with FINMA’s statements from the consultation, including on data encryption as a protective measure.

3.  [Banking Act (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 on banking secrecy.

4.  [Swissbanking: Banking secrecy and cloud, legal opinion 2019](https://www.swissbanking.ch/_Resources/Persistent/b/1/3/3/b13339fe5dcd344e715dbe0a899afe3c8c56ed0b/SBVg_Bankgeheimnis_und_Cloud_Gutachten_LLAG_2019_DE.pdf): Legal opinion on the interpretation of Art. 47 of the Banking Act, including customer consent.

5.  [Federal Act on Data Protection (Data Protection Act)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 on data security, Art. 24 on reporting data security breaches.

6.  [Data Protection Ordinance](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Arts. 1 to 6 on data security principles and protection objectives.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Types of electronic signatures and seals, requirements for qualified certificates.

8.  [Code of Obligations](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 para. 2bis on the equivalence of qualified electronic signatures to handwritten signatures, Art. 958f on retention.

9.  [Civil Procedure Code](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 on free assessment of evidence, Art. 177 on the definition of documents.

10.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSign certificate profiles (as of September 18, 2025), assignment of Silver, Gold, and Organization Validated to validation types and subject fields.

11.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Binding rules for publicly trusted S/MIME certificates, definition of validation types.
