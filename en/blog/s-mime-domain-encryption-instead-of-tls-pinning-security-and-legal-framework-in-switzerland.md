---
title: "S/MIME Domain Encryption Instead of TLS Pinning: Security and Legal Framework in Switzerland"
navTitle: "Domain Instead of Pinning"
description: "Why S/MIME domain encryption protects email traffic between two organizations better than TLS pinning, what an attacker can do with an intercepted S/MIME message, how the DPA, DPO and professional secrecy affect the choice, and why MTA-STS and DANE take over transport protection."
date: "2026-11-16"
kategorie: "Email Encryption"
timeToRead: "10 min read"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-domain-encryption-instead-of-tls-pinning-security-and-legal-framework-in-switzerland"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/en/blog/s-mime-domain-encryption-instead-of-tls-pinning-security-and-legal-framework-in-switzerland
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T15:56:59.714Z
translationReview: automatic
---

# S/MIME Domain Encryption Instead of TLS Pinning: Security and Legal Framework in Switzerland

In brief: TLS pinning protects the connection between two mail servers, while S/MIME domain encryption protects the content of a message throughout its entire journey between two organizations. If you have to choose between the two, domain encryption is the better choice from both a technical and legal perspective. MTA-STS and DANE provide the transport protection previously delivered by pinning.

In a current project, an existing mail connection between two organizations that has so far been secured with enforced TLS and certificate pinning is to be migrated to S/MIME domain encryption through a secure mail service. The service cannot use both for the same connection at the same time: either pinning applies or domain encryption does. The question of whether this results in lost security is justified. The answer is clear once you consider what each method protects.

**Switzerland focus:** The legal assessment follows Swiss law, in particular the DPA, DPO and SCC. Where EU rules differ, this is indicated in a labeled note. This article is not a substitute for legal advice in individual cases.

## What TLS pinning protects

Without additional measures, TLS between mail servers is opportunistic. This creates three attack vectors:

- **Downgrade:** An attacker in the transmission path removes the STARTTLS offer from the SMTP session. The sending server then delivers in plaintext.
- **Any certificate:** Sending mail servers do not verify the recipient's certificate by default. A server with a self-issued certificate is accepted.
- **Spoofed MX:** Anyone who manipulates the DNS response redirects the email to their own server, which offers proper TLS.

Pinning closes all three gaps for a specific partner. The sending gateway enforces TLS and accepts only a specified certificate or key. However, protection ends with the connection. On the receiving server, in queues, spam and virus filters, journaling and archiving systems, the message is available in plaintext. If it passes through several servers, pinning applies only to the segment for which it was configured.

There is also the operational aspect. The pin is a manual rule on the sender's side. If the recipient renews its certificate or changes its routing, for example by adding a new filtering service or a backup MX, delivery stops until the rule is updated. In the event of a disruption, the rule is therefore often temporarily relaxed and not always re-enabled.

## What S/MIME domain encryption protects

With domain encryption, the sender's gateway encrypts the message using the public key from the recipient domain's domain certificate. Only the recipient's gateway decrypts it again. All systems in between see only the encrypted content, regardless of how many servers the message passes through or where it is buffered. In addition, the sender's gateway signs the message with its own domain certificate.

| Protection | TLS pinning | S/MIME domain encryption |
|---|---|---|
| Email body and attachments | only during transmission over the pinned segment | throughout the entire journey, including intermediary servers and queues |
| Subject, sender, recipient, date | encrypted during transmission | readable unless header protection is enabled |
| Envelope, size, time | encrypted during transmission | readable |
| Authenticity of the receiving server | yes, an incorrect server is rejected | no, the email is handed over to the server provided by MX routing |
| Forward secrecy | yes, with ECDHE key exchange | no |
| Integrity and origin after delivery | no | yes, through a signature |
| Dependency on routing and infrastructure | high, the pin applies to one server or key | low, protection is tied to the domain certificate |

Pinning therefore has three properties that domain encryption does not offer: it protects metadata, prevents an incorrect server from accepting the email, and provides forward secrecy. For content, domain encryption is clearly superior.

## What an attacker can do with an intercepted S/MIME message

Assume an attacker places their own mail server in the transmission path, through spoofed MX records, BGP hijacking, or a compromised relay.

They cannot do the following:

- **Read content:** Text and attachments are encrypted with the recipient domain's public key. Without the recipient gateway's private key, they remain unreadable.
- **Modify content without detection:** If the message is signed and the recipient gateway requires a valid signature, any modification is detected.
- **Create messages in the sender's name:** They lack the sender domain's private signing key.

They can do the following:

- **Analyze metadata:** Sender, recipient, time, size, and usually the subject line are readable. A subject line such as “Discharge report Max Muster” already constitutes a disclosure of personal data.
- **Store messages and decrypt them later:** S/MIME does not provide forward secrecy. If the private domain key is later compromised, all stored messages become readable. The same applies in the long term if RSA becomes vulnerable to quantum computers.
- **Suppress or delay messages:** The sender will not necessarily notice. Availability is affected, which is relevant for deadlines.
- **Modify unprotected headers:** For example, they can rewrite the subject or `Reply-To` to redirect replies. The encrypted content remains unchanged.
- **Deliberately manipulate ciphertext:** For messages that are only encrypted and not signed and use AES-CBC, ciphertext can be modified in such a way that parts of the plaintext may leak after decryption. This is the principle behind the 2018 EFAIL attacks. Mandatory signature verification or authenticated encryption with AES-GCM under S/MIME 4.0 prevents this.

Intercepted plaintext with transport encryption alone is a complete data leak. An intercepted S/MIME message primarily affects metadata and availability.

## Legal assessment

No Swiss law mandates S/MIME, TLS pinning, or any other specific method. The requirements are technology-neutral and define protection objectives. However, the choice of method must be assessed against these objectives and documented.

| Regulation | Content | Significance for the choice of method |
|---|---|---|
| DPA Art. 8, DPO Arts. 1 to 3 | appropriate data security, protection objectives, transport control | protection during disclosure, measured against risk and the state of the art |
| DPA Art. 61(c) | fine for violating minimum data security requirements | personal criminal liability of the responsible person |
| DPA Art. 9 | processing by processors | every system with access to plaintext must be considered |
| DPA Arts. 16 and 17 | disclosure abroad | processing of plaintext on foreign systems |
| DPA Art. 24 | notification of data security breaches | risk to affected individuals depends on whether leaked data is readable |
| SCC Art. 321, DPA Art. 62, ATSG Art. 33 | professional secrecy, professional duty of confidentiality, confidentiality in social insurance | content must be protected against any unauthorized access |
| CPC Art. 177 | electronic files as records | signature strengthens proof of origin and integrity |

### Data security under the DPA and DPO

Art. 8 DPA requires data security appropriate to the risk. The DPO specifies the protection objectives of confidentiality, availability, integrity, and traceability and names transport control in Art. 3 as one measure: personal data may not be read, copied, modified, or deleted without authorization during disclosure. Correspondingly higher requirements apply to sensitive personal data under Art. 5(c) DPA, such as health data or data concerning social assistance measures.

TLS pinning fulfills transport control for one segment of transmission. However, the message is stored on several systems before reaching its destination, where it can be read in plaintext. With domain encryption, content is protected across the entire route between the two organizations. The signature additionally covers the protection objective of integrity, which TLS can no longer demonstrate after delivery.

### Personal criminal liability

Under Art. 61(c) DPA, anyone who intentionally fails to comply with the minimum data security requirements under Art. 8(3) DPA is liable, upon complaint, to a fine of up to CHF 250,000. The fine is imposed on the responsible natural person, not the company. Content encryption based on the current state of the art and documented in a risk assessment provides stronger evidence for these persons than a transport rule for individual servers.

**EU note:** Art. 32 GDPR likewise requires appropriate technical and organizational measures and expressly mentions encryption. However, fines under Art. 83 GDPR are imposed on the company and may amount to up to EUR 10 million or 2% of worldwide annual revenue for violations of Art. 32.

### Processors and disclosure abroad

Every system along the path that processes plaintext must be legally considered: filtering services, relays, cloud gateways, archiving services. If third parties operate these systems, this is generally processing by a processor under Art. 9 DPA, with the corresponding contracts and oversight obligations. If the systems are located abroad, the rules on disclosure abroad under Arts. 16 and 17 DPA also apply.

With domain encryption, these systems process only ciphertext. The group of entities that can access content is limited to the two gateways. However, metadata remains personal data, and the stated obligations continue to apply to it.

### Notification obligation for data security breaches

A data security breach likely to result in a high risk to the affected individuals must be reported to the FDPIC under Art. 24 DPA. If messages leak through a compromised intermediary server, the assessment depends on whether their contents are readable. If they are strongly encrypted and the keys are not affected, the risk to affected individuals is generally low, provided no sensitive information appears in the subject line. With transport encryption alone, the attacker has the content in plaintext.

### Professional secrecy and duties of confidentiality

Where data is subject to professional secrecy under Art. 321 SCC, for example for physicians, attorneys, or their assistants, or to a statutory duty of confidentiality such as Art. 33 ATSG in the social insurance sector, the content must be protected against any unauthorized access. This includes operators of intermediary systems. Art. 62 DPA also criminalizes violation of professional confidentiality by persons not covered by Art. 321 SCC. Only content encryption meets this requirement.

### Evidentiary value of the signature

Under Art. 177 CPC, electronic files are records whose evidentiary value is freely assessed by the court. A valid S/MIME signature from the sender domain demonstrates that the message originated from that domain and has not been modified since signing. This strengthens evidence for deadlines, commitments, or instructions. It does not replace a signature; the formal requirements of the CO are met only by a qualified electronic signature under ZertES. With TLS pinning, neither origin nor integrity can be demonstrated after delivery.

## MTA-STS and DANE as replacements for TLS pinning

If pinning is removed, the properties provided only by transport protection are missing: enforced TLS, an authenticated receiving server, and thus protection of metadata from eavesdropping and redirection. MTA-STS and DANE provide exactly these properties, as a standard that the recipient publishes once for all senders.

**MTA-STS (RFC 8461):** The recipient publishes a policy with the permitted MX host names and the `https://mta-sts.<domain>/.well-known/mta-sts.txt` mode `enforce`. A sender that supports MTA-STS retrieves the policy, caches it, and delivers only via TLS with a valid certificate to one of the listed MX hosts. A spoofed MX or a server without a valid certificate for the MX name is rejected.

**DANE (RFC 7672):** The recipient publishes a TLSA record containing the hash of its certificate or key in its DNSSEC-signed zone. The sender verifies the MX certificate against this record. Technically, DANE is pinning that the recipient publishes in DNS and protects with DNSSEC.

| | TLS pinning | MTA-STS | DANE |
|---|---|---|---|
| Maintained by | sender, manually for each partner | recipient, once for all senders | recipient, once for all senders |
| Applies to | one configured partner | all senders supporting MTA-STS | all senders supporting DANE |
| Certificate renewal | pin must be updated by the sender | no adjustment needed as long as the new certificate is valid | no adjustment needed with the same key; otherwise publish the new TLSA record in advance |
| Trust basis | exactly this certificate or key | public certificate authorities | DNSSEC |
| Weakness | operational effort, relaxation during disruptions | first retrieval and expired cache (Trust on First Use), misissuance by a CA | requires DNSSEC |
| Error reports | none | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

For documentation under Art. 8 DPA, MTA-STS and DANE have another advantage: TLS-RPT gives the recipient daily reports on whether deliveries were encrypted or failed. Pinning provides no such feedback. The article [DNS Records for Email](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) describes how to configure both methods in the section on MTA-STS, TLS-RPT, and DANE.

## Who must check MTA-STS and DANE

The system that hands the message directly to the recipient's MX must always perform the checks. The operator of the MX must publish the records.

With a secure mail service, the outbound path typically looks like this:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Pinning has so far protected the final segment. Here, the service must check the partner's MTA-STS and DANE; your own mail server has no influence over this segment. You secure the segment from your own mail server to the service yourself, with enforced TLS and certificate verification on the connector.

On the receiving side, it depends on the MX. If the MX points to the service, the service must provide matching certificates, and you publish MTA-STS and TLSA records that match its servers. For DANE, this requires the service to announce key changes in time. If the MX points to your own gateway, such as Exchange Online, configure MTA-STS and DANE there. If both organizations use the same service, it may deliver the message within its own platform; in that case, there is no public SMTP segment between the two organizations.

If the service supports neither MTA-STS nor DANE for sending, the content remains protected by S/MIME. Protection of metadata and authentication of the receiving server on the final segment then remain open. This residual risk belongs in the risk assessment.

## Residual risks and measures

| Residual risk | Measure |
|---|---|
| Personal data in the subject line is readable | Rule: no personal data in subject lines, agreed in writing with the partner |
| No forward secrecy | Store the private domain key as non-exportable, ideally in an HSM; log access |
| Manipulation of unsigned messages | Mandatory signatures on both sides, AES-GCM where both gateways support it |
| Plaintext delivery in the event of errors | Hold messages when encryption fails rather than delivering them in plaintext |
| Metadata in transit | MTA-STS in `enforce` mode and, where DNSSEC is available, DANE; evaluate TLS-RPT |
| Expired certificates | Mutual notification at least 30 days before expiration, verify fingerprints through a second channel |

The decision to use domain encryption and the remaining risks should be included in the data security concept or in the documentation of processing activities.

## Questions for the gateway service provider

1.  Does the service check MTA-STS policies in `enforce` mode and DANE records for the recipient domain when sending?

2.  Which servers deliver messages, and can TLS-RPT reports or delivery logs for transport encryption be reviewed?

3.  As an MX, does the service support MTA-STS and DANE for incoming messages, and how are key changes announced?

4.  Can the connector from your own mail server to the service be operated with enforced TLS and certificate verification?

5.  Does delivery between two customers of the service occur via public SMTP or within the platform?

6.  Which algorithms are used for encryption and signing, and is AES-GCM supported?

## Sources

1.  [Federal Act on Data Protection (DPA)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 8 on data security, Art. 9 on processing by processors, Arts. 16 and 17 on disclosure abroad, Art. 24 on notification obligations, Arts. 61 and 62 on criminal provisions.

2.  [Data Protection Ordinance (DPO)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): Arts. 1 to 3 with principles, protection objectives, and technical and organizational measures, including transport control.

3.  [Swiss Criminal Code (SCC)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): Art. 321 on the violation of professional secrecy.

4.  [Federal Act on the General Part of the Social Insurance Law (ATSG)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): Art. 33 on the duty of confidentiality of implementing bodies.

5.  [Civil Procedure Code (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 177 on the concept of records, including electronic files.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Specification of the policy, retrieval rules, and `enforce` mode.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE for SMTP, TLSA records, and verification rules for sending servers.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Format and delivery of TLS-RPT reports.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): Current S/MIME specification with AuthEnvelopedData and AES-GCM.

10.  [EFAIL](https://efail.de/): Description of the attacks published in 2018 against unsigned S/MIME and OpenPGP messages in CBC mode.

11.  [General Data Protection Regulation (GDPR)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): Art. 32 on security of processing, Art. 83 on fines (EU note).
