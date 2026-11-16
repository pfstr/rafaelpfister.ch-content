---
title: "S/MIME for Banks: Silver or Gold Certificate from SwissSign?"
navTitle: "S/MIME Silver/Gold"
description: "A bank that signs and encrypts customer emails with S/MIME needs Gold certificates for personal mailboxes and Organization Validated certificates for functional mailboxes. Silver encrypts just as strongly, but does not identify the bank as the sender. What SwissSign verifies, what the certificate contains, and which requirements apply in Switzerland."
date: "2026-11-16"
kategorie: "Email Encryption"
timeToRead: "9 min read"
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
translationSourceHash: f95529dd7c72d4d783714affbcf18d96e37adf66a52379b94212034dda610b74
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T14:47:26.015Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/s-mime-with-banks-swisssign-silver-or-gold-certificate
---

# S/MIME for Banks: Silver or Gold Certificate from SwissSign?

In brief: A bank that signs and encrypts emails to customers and business partners with S/MIME should equip personal mailboxes with Gold certificates and functional mailboxes with Organization Validated certificates from SwissSign. Silver encrypts just as strongly, but the certificate contains only the email address, not the bank's name. Neither the law nor FINMA specifies a certificate level.

The question of which certificate level is required for sending email arises regularly in my projects with financial institutions. It is usually raised at the start of a gateway project or when business customers require confidential documents to be sent only in encrypted form. To answer it, two functions of S/MIME need to be distinguished: encryption, which protects the content, and the signature, which proves to the recipient who sent the email.

**Switzerland focus:** This article covers the legal situation in Switzerland. Where EU rules differ, this is indicated in a labeled note.

## What SwissSign verifies for Silver and Gold

Since September 1, 2023, the `S/MIME Baseline Requirements` of the CA/Browser Forum have applied to publicly trusted S/MIME certificates. They distinguish four validation types, depending on what the certification authority verifies before issuance. SwissSign assigns its products to these types in its published certificate profiles (CPR S/MIME):

| Product | Type under CA/B Forum | What is verified | What the certificate contains |
|---|---|---|---|
| E-Mail ID Silver | `Mailbox-validated` | Control over the mailbox | email address only |
| E-Mail ID Gold | `Sponsor-validated` | Organization and individual | First and last name (or pseudonym), organization, organization identifier, country, email address |
| Organization Validated | `Organization-validated` | Organization | Organization name, no individual |

For Silver, SwissSign sends a confirmation code to the requested address. Anyone able to receive the email is issued the certificate. The identity of the individual and the organization are not verified, and the address also appears in the certificate as `Common Name`.

At SwissSign, Gold is offered exclusively as `Sponsor-validated`. SwissSign verifies the organization, and the individual is associated with it. The individual's identity is verified either by SwissSign or by the organization itself as the registration authority, generally as part of a Managed PKI. For functional mailboxes such as `kundendienst@` or `zahlungsverkehr@`, the `Organization-validated` profile is intended; the bank appears as the organization in the certificate, but no individual does.

The levels do not differ cryptographically. All profiles use RSA keys of at least 2048 bits, the key usages `digitalSignature` and `keyEncipherment` as well as the extended usage `id-kp-emailProtection`.

## What the difference means for a bank as the sender

| | Silver | Gold | Organization Validated |
|---|---|---|---|
| Encrypting and decrypting emails | yes | yes | yes |
| Signature displayed in the email client | Email address | Name and bank | Bank |
| What the signature asserts | Sender has access to the mailbox | Verified individual at this bank | Mailbox of this bank |
| Use at a bank | Unsuitable for customer communications | Personal mailboxes | Functional mailboxes |

The difference concerns the signature. A bank signs outgoing emails so recipients can verify that the message genuinely comes from it and has not been altered in transit. With Silver, this is only possible to a limited extent:

- Anyone who registers a similarly named domain, such as `musterbank-ch.com` instead of `musterbank.ch`, can obtain a Silver certificate for addresses on that domain without further verification. Their phishing emails are then signed just as validly as the bank's genuine emails, and in both cases the certificate contains only an email address.
- Anyone who takes over a bank mailbox can also request a Silver certificate for that address.

A Gold or Organization Validated certificate contains the verified name of the bank in the `Organization` field. Only those that pass SwissSign's organization verification receive this entry. For recipients who view the signature, this is the relevant proof.

For encryption, the level is irrelevant. An email encrypted with Silver is just as well protected as one encrypted with Gold.

## Encryption requires recipients' certificates

To encrypt an email with S/MIME, the bank needs the recipient's public key. The bank's own certificate is not used for this; it serves to sign emails and receive encrypted replies.

Most retail customers do not have an S/MIME certificate. Banks therefore usually deliver confidential emails to them through an encryption gateway, such as SEPPmail (GINA) or Kiteworks/Totemomail, which provides the message in a web portal or as an encrypted attachment. S/MIME encryption is used primarily for business customers and partners that operate their own certificates or a gateway. When both sides use a gateway, domain certificates or enforced TLS between the mail servers are often sufficient.

## Certificates in operation: Managed PKI and gateway

For more than a handful of mailboxes, a bank obtains its certificates through SwissSign's Managed PKI. The bank acts as the registration authority and confirms the identity of its employees itself; this corresponds to type `Sponsor-validated`. Gateways such as SEPPmail and Kiteworks/Totemomail can use an interface to the Managed PKI to request, renew, and assign certificates to mailboxes automatically.

Three points from projects are known to cause operational problems:

- Gold requests for functional mailboxes fail because the gateway imports first and last names from the directory and SwissSign rejects generic terms such as “Support” or “Helpdesk” in the `GivenName` field. For such mailboxes, the `Organization-validated` profile can be configured in the gateway.
- Automatic renewals can fail for weeks without being noticed, until the first certificates expire and customers can no longer send encrypted emails. The expiration dates of issued certificates and error messages from renewal runs should therefore be monitored.
- The gateway signs its requests to the Managed PKI with its own registration-authority certificate. If it expires, all automation stops.

## Legal requirements in Switzerland

No Swiss law mandates S/MIME, and none distinguishes between certificate levels. A bank's obligations concern the protection of customer data; it decides for itself how to implement this.

| Regulation | Content | Significance for the bank's email delivery |
|---|---|---|
| Banking Act, Art. 47 | Banking secrecy | Customer data must not be disclosed without authorization; unencrypted transmission over the internet is a risk the bank must assess |
| FINMA Circular 2023/1 | Protection of critical data | Protection also during transmission, technology-neutral, no requirement for certificates |
| FADP Art. 8 and DPO Art. 1 through 6 | Data security appropriate to the risk | Encryption is a common measure; no specific method is prescribed |
| ZertES and CO Art. 14 para. 2bis | Only the qualified electronic signature is equivalent to a handwritten signature | S/MIME signatures do not replace a signature |

### Banking secrecy and FINMA

Banking secrecy under Art. 47 of the Banking Act obliges the bank, its governing bodies, and its employees not to disclose customer data without authorization. FINMA Circular 2023/1, “Operational Risks and Resilience: Banks,” requires banks to protect their critical data throughout its lifecycle, including during transmission. The bank defines how this is done in its own policies. The choice of certificate level is therefore part of the internal policy and should be documented there.

### Data protection: FADP and DPO

Art. 8 FADP requires controllers to ensure data security appropriate to the risk. The Data Protection Ordinance specifies this through protection objectives such as confidentiality and integrity, but does not name any procedures. S/MIME, a gateway with webmail delivery, or an exchange platform meet this requirement equally, provided they are used appropriately for the risk.

### Electronic signature: ZertES and CO

ZertES distinguishes between electronic, advanced, regulated, and qualified electronic signatures. Under Art. 14 para. 2bis CO, only the qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature. SwissSign's S/MIME certificates are issued under the rules of the CA/Browser Forum and the ETSI policies LCP (Silver) and NCP (Gold), and are not qualified certificates under ZertES. Contracts or declarations for which written form is required cannot be validly executed through an S/MIME-signed email, including with Gold.

**EU note:** In the EU, under Art. 25 para. 2 eIDAS, only the qualified electronic signature is likewise equivalent to a handwritten signature.

## Recommendation

- **Personal mailboxes** for customer advisors and other employees with customer contact: Gold (`Sponsor-validated`) through the Managed PKI.
- **Functional mailboxes:** `Organization-validated`, so the bank appears as the sender in the certificate.
- Do not use **Silver** for customer communications because the signature does not identify the bank as the sender.
- **Encryption for retail customers** through the gateway's push or portal delivery; S/MIME for recipients with their own certificate or gateway.
- **Signature and sender verification** complement each other: The S/MIME signature does not replace SPF, DKIM, and DMARC, but it displays the verified organization to the recipient in the email client.
- **Customer fact sheet:** Describe which certificates the bank uses, how customers can verify a signature, and which certificate levels the bank accepts from customers who want to send encrypted emails themselves. If no such fact sheet exists, the question goes to customer advisors, who usually cannot answer it.

## Sources

1.  [SwissSign CPR S/MIME: Certificate, CRL and OCSP Profiles for S/MIME Certificates](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSign certificate profiles (as of September 18, 2025), assignment of Silver, Gold, and Organization Validated to validation types, Subject fields, and policy OIDs.

2.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): binding rules for publicly trusted S/MIME certificates, definition of the four validation types.

3.  [SEPPmail: Managed PKI SwissSign](https://docs.seppmail.com/en/07_mi_10_mpki_11_swisssign.html): connection of the SEPPmail gateway to SwissSign's Managed PKI.

4.  [Banking Act (BankG)](https://www.fedlex.admin.ch/eli/cc/51/117_121_129/de): Art. 47 on banking secrecy.

5.  [FINMA: Circulars](https://www.finma.ch/de/dokumentation/rundschreiben/): overview including Circular 2023/1, “Operational Risks and Resilience: Banks,” in force since January 1, 2024.

6.  [Federal Act on Data Protection (FADP)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 on data security.

7.  [Data Protection Ordinance (DPO)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Art. 1 through 6 on the principles and protection objectives of data security.

8.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): types of electronic signature and requirements for qualified certificates.

9.  [Code of Obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 para. 2bis on the equivalence of the qualified electronic signature to a handwritten signature.
