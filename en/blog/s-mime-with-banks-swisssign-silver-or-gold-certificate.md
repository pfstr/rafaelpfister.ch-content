---
title: "S/MIME with banks: SwissSign Silver or Gold certificate?"
navTitle: "S/MIME Silver/Gold"
description: "For encrypted emails with a bank, a SwissSign Silver certificate is technically sufficient. Gold also verifies the sender's name and organization. Which level verifies what, what appears in the certificate, and which legal requirements apply in Switzerland."
date: "2026-11-16"
kategorie: "Email Encryption"
timeToRead: "8 min read"
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
url: https://rafaelpfister.ch/en/blog/s-mime-with-banks-swisssign-silver-or-gold-certificate
translationSourceHash: cbcb890de323f2b9db40a5682192296524250e7bc8c23a67c6f643bcba605a14
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:03:11.245Z
translationReview: required
---

# S/MIME with banks: SwissSign Silver or Gold certificate?

In brief: Neither Swiss law nor FINMA mandates a specific certificate level for S/MIME. A Silver certificate is fully sufficient for encryption from a technical perspective. Gold additionally verifies the sender's name and organization, making it the appropriate choice if the bank does not specify a requirement.

The question of whether a SwissSign Silver or Gold certificate is needed for email communication with a bank comes up regularly in my projects. Usually, the bank has no fact sheet on the matter, and the customer advisor cannot answer the question. The answer depends on what the certificate is used for: encryption or proof of who sent an email.

**Switzerland focus:** This article covers the legal situation in Switzerland. Where EU rules differ, this is indicated accordingly.

## What SwissSign verifies for Silver and Gold

Since September 1, 2023, publicly trusted S/MIME certificates have been subject to the `S/MIME Baseline Requirements` of the CA/Browser Forum. They distinguish between four validation types, depending on what the certification authority verifies before issuance. SwissSign assigns its products to these types in the published certificate profiles (CPR S/MIME):

| Product | CA/B Forum type | What is verified | What appears in the certificate |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control over the mailbox | Email address only |
| E-Mail ID Gold | `Sponsor-validated` | Organization and person | First and last name (or pseudonym), organization, organization identifier, country, email address |
| Organization Validated | `Organization-validated` | Organization | Organization name, no individual |

For Silver, SwissSign sends a confirmation code to the requested address. Anyone who can receive the email obtains the certificate. The person's identity is not verified, and the certificate also lists the address as the `Common Name`.

SwissSign lists Gold exclusively as `Sponsor-validated`. The organization is verified, and the person is associated with it: their identity is verified either by SwissSign or by the organization itself as the registration authority, for example as part of a managed PKI. The certificate profiles do not list a profile for private individuals without an organization (`Individual-validated`).

For functional mailboxes such as `support@` or `buchhaltung@`, the `Organization-validated` profile is intended. Gold applications for such mailboxes may fail because SwissSign rejects generic terms such as “Support” or “Helpdesk” in the `GivenName` field.

The levels do not differ cryptographically. All profiles use RSA keys of at least 2048 bits, the `digitalSignature` and `keyEncipherment` key usages, and the `id-kp-emailProtection` extended usage. An email encrypted with Silver is protected just as well as one encrypted with Gold.

## What the difference means in practice

| | Silver | Gold |
|---|---|---|
| Encrypting and decrypting emails | yes | yes |
| Signature display in Outlook | Email address | Name and organization |
| What the signature indicates | Sender has access to the mailbox | Sender is a verified person at that organization |
| Effort required to apply | Confirmation by email | Proof of organization and identity verification |

The difference becomes apparent in the event of an account takeover. Anyone who controls a mailbox can apply for a Silver certificate for that address and use it to send validly signed emails. A Gold certificate, on the other hand, requires the organization and the person to be verified. For a bank that accepts payment orders or contractual correspondence by email, for example, a Gold signature therefore carries more weight.

This does not matter for encryption. The bank only needs the recipient's public key, and it is equivalent at both levels.

## Legal requirements in Switzerland

No Swiss law requires S/MIME, and none distinguishes between certificate levels. There are general obligations to protect data, but their technical implementation remains open.

| Regulation | Content | Relevance to S/MIME |
|---|---|---|
| DPA Art. 8 and DPO Art. 1 to 6 | Data security appropriate to the risk | Encryption is a common measure for sensitive personal data; no specific method is required |
| Banking Act Art. 47 | Banking secrecy | Obligates the bank, not its customers |
| FINMA Circular 2023/1 | Protection of critical data at banks | Technology-neutral; no certificate requirements |
| ZertES and CO Art. 14 para. 2bis | Only the qualified electronic signature is equivalent to a handwritten signature | Silver and Gold are not qualified signatures |
| VeÜ-ZSSV | Electronic submissions to courts and debt enforcement offices | Qualified signature and recognized delivery platform required; S/MIME is insufficient |

### Data protection: DPA and DPO

Article 8 of the DPA requires controllers and processors to ensure data security appropriate to the risk. The Data Protection Ordinance specifies this through protection objectives such as confidentiality and integrity, but does not prescribe any particular methods. Whether someone uses S/MIME, a gateway with webmail delivery, or an exchange platform is an assessment to be made on a case-by-case basis.

### Banking secrecy and FINMA

Banking secrecy under Article 47 of the Banking Act and FINMA Circular 2023/1, “Operational Risks and Resilience: Banks,” are addressed to the bank. The circular requires banks to protect their critical data throughout its entire lifecycle, including during transmission. Banks set out how they implement this in their own policies. Whether Silver is accepted is therefore a question of the individual bank's internal policy.

### Electronic signature: ZertES and CO

ZertES distinguishes between electronic, advanced, regulated, and qualified electronic signatures. Under Article 14 para. 2bis of the CO, only a qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature. SwissSign S/MIME certificates are issued under the CA/Browser Forum rules and the ETSI LCP (Silver) and NCP (Gold) policies and are not qualified certificates under ZertES.

A contract for which written form is required therefore cannot be validly concluded with an S/MIME-signed email, not even with Gold. In a dispute, a Gold signature may nevertheless serve as evidence; the court assesses its weight freely.

### Submissions to courts and authorities

For electronic submissions in civil, criminal, debt enforcement, and bankruptcy proceedings, VeÜ-ZSSV requires a qualified electronic signature and transmission via a recognized delivery platform. An S/MIME email does not meet these requirements.

**EU note:** Under Article 25(2) eIDAS, only the qualified electronic signature is likewise equivalent to a handwritten signature in the EU.

## If the bank does not specify a requirement

If no fact sheet is available, the situation can be clarified with three questions for the bank:

1.  Does the bank require a certificate with verified identity for signed emails, or is email-only validation sufficient?

2.  Does the bank use an encryption gateway (e.g., SEPPmail or Kiteworks/Totemomail) with a domain certificate, or personal certificates for its employees?

3.  How does the bank provide its own public keys?

A short inquiry to customer service that can be forwarded to IT:

> I am obtaining an S/MIME certificate for encrypted email communication with you. Please let me know whether you require a certificate with verified identity (e.g., SwissSign E-Mail ID Gold with name and organization), or whether a certificate with email-only validation (Silver) is sufficient. I would also appreciate your public certificate or information on whether you use a gateway with a domain certificate.

Many banks send confidential emails through a gateway with webmail delivery anyway. The recipient then logs in to the portal and does not need their own certificate. If both sides use a gateway, domain certificates or enforced TLS between the mail servers are often sufficient, and personal certificates are not needed.

If the question remains unanswered, Gold is the appropriate choice for business correspondence. Gold covers encryption and signing, and switching from Silver to Gold later means applying again and redistributing the public key to all counterparties. Silver is sufficient if the bank confirms that it only encrypts and does not evaluate signatures. An `Organization-validated` certificate is intended for functional mailboxes.

## Sources

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSign certificate profiles (as of September 18, 2025), assignment of Silver, Gold, and Organization Validated to validation types, subject fields, and policy OIDs.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): binding rules for publicly trusted S/MIME certificates, definition of the four validation types.

3.  [Federal Act on Data Protection (DPA)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Article 8 on data security.

4.  [Data Protection Ordinance (DPO)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Articles 1 to 6 on the principles and protection objectives of data security.

5.  [Banking Act](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Article 47 on banking secrecy.

6.  [FINMA: Circulars](https://www.finma.ch/de/dokumentation/rundschreiben/): overview including Circular 2023/1, “Operational Risks and Resilience: Banks,” in force since January 1, 2024.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types of electronic signatures and requirements for qualified certificates.

8.  [Code of Obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Article 14 para. 2bis on the equivalence of the qualified electronic signature to a handwritten signature.
