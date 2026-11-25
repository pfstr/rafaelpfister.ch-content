---
title: "S/MIME with an Organization Entry: organizationIdentifier and UID in the Certificate"
navTitle: "S/MIME with an Organization"
description: "What an S/MIME certificate with an organization entry contains, how the UID is included in the certificate as an organizationIdentifier, and what the certificate authority verifies for companies and individuals. Also includes a comparison of SwissSign and DigiCert, commands for checking the certificate type, and a distinction from ZertES."
date: "2026-12-15"
kategorie: "Email Encryption"
timeToRead: "13 min read"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-with-an-organization-entry-organizationidentifier-and-uid-in-the-certificate"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:54:01.304Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/s-mime-with-an-organization-entry-organizationidentifier-and-uid-in-the-certificate
---

# S/MIME with an Organization Entry: organizationIdentifier and UID in the Certificate

Certificate authorities (CAs) distinguish between S/MIME certificates with and without an organization entry. In addition to the email address, a certificate with an organization entry identifies the company to which the mailbox belongs: with its verified name and a registration number. For Swiss companies, this registration number is the UID, i.e., the CHE number from the commercial register.

In short: An organization entry is available only for the certificate types `Organization-validated` and `Sponsor-validated` in the S/MIME Baseline Requirements. It consists of the company name in the `O` field and the `organizationIdentifier` field (OID 2.5.4.97), for a Swiss company, for example, `NTRCH-CHE-123.456.789`. The CA verifies the company in an official register before entering this information in the certificate. A mailbox-validated certificate, by contrast, proves only access to a mailbox. At SwissSign, this corresponds to choosing between Silver and Gold; at DigiCert, to choosing between “Secure Email for Individual Mailbox” and the “for Employee” and “for Organization” products. An organization entry does not create a qualified signature under ZertES.

**Switzerland focus:** The legal classification follows Swiss law (ZertES, OR). Where EU rules differ, this is noted accordingly.

## The Four Certificate Types

Since September 1, 2023, the `S/MIME Baseline Requirements` (BR) of the CA/Browser Forum have applied to publicly trusted S/MIME certificates. They distinguish four certificate types according to who is listed as the holder (`Subject`) in the certificate:

| Type | Holder in the certificate | Organization entry | What the CA verifies |
|---|---|---|---|
| `Mailbox-validated` (MV) | email address only | not permitted | Control of the mailbox or email domain |
| `Organization-validated` (OV) | an organization, no individual | required | Organization and mailbox or email domain |
| `Sponsor-validated` (SV) | an individual affiliated with an organization | required | As for OV, plus the individual; this is verified by the CA or by the organization itself |
| `Individual-validated` (IV) | an individual without an organization | not permitted | Identity of the individual and mailbox |

“S/MIME with an organization entry” refers to the OV and SV types. For both, the BR require the `organizationName` and `organizationIdentifier` fields in all three generations: `Legacy`, `Multipurpose`, and `Strict`. The generations differ in the permitted Subject fields, extensions, and intended uses. The BR discontinued the `Legacy` generation as of July 15, 2025; SwissSign and DigiCert have not issued it since then.

OV certificates are suitable for functional mailboxes such as `info@` or `buchhaltung@`, which are not assigned to an individual. SV certificates are intended for employees’ personal mailboxes and additionally contain the person’s name or a pseudonym.

## Structure of the organizationIdentifier

The `organizationIdentifier` field (OID 2.5.4.97) contains an organization registration number. Certificates under eIDAS and ZertES use the same field. For S/MIME certificates, the BR prescribe the following structure:

| Part | Example | Meaning |
|---|---|---|
| Registration scheme, 3 characters | `NTR` | `National Trade Register`, the commercial register |
| Country code according to ISO 3166-1 | `CH` | Country in which the register is maintained |
| Hyphen | `-` | Separator |
| Registration number | `CHE-123.456.789` | Organization’s number in the register |

Combined, this results in `NTRCH-CHE-123.456.789`. Because the UID itself contains a hyphen, the BR specify that the scheme and country code must never contain a hyphen. The first hyphen from the left therefore separates the scheme from the registration number; all subsequent hyphens belong to the number. If a register assigns numbers only at the state or provincial level, a `+` followed by the region code comes after the country code, for example `NTRUS+CA-12345678` for California.

The BR recognize the following registration schemes:

| Scheme | Origin of the number | Example from the BR |
|---|---|---|
| `NTR` | Commercial or business register | `NTRGB-12345678` |
| `VAT` | Tax authority (VAT number) | `VATEL-123456789` |
| `PSD` | Authorization number of a payment service provider under EU Directive 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` under ISO 17442, country code always `XG` | `LEIXG-` followed by the LEI |
| `GOV` | Government bodies without a number from one of the other schemes | `GOVUS` |
| `INT` | International organizations | `INTXG` |

For Swiss companies, `NTR` with the UID is the standard case. The UID is assigned by the Federal Statistical Office. Commercial registers use it as the company number; it also serves as the VAT number and remains the same if a company moves its registered office to another canton. A VAT number, by contrast, may cease to apply when tax liability ends, and not every organization has one. SwissSign adopts the UID including the periods, as it appears in the UID register, and gives `NTRCH-CHE-109.357.012` as the example for SwissSign AG.

**EU note:** In Germany, local courts assign commercial register numbers, and the same number can occur at multiple courts. Since version 1.0.9 of May 2025, the BR have therefore permitted the `European Unique Identifier` (EUID) in the `NTR` scheme and recommend it for German companies, for example `NTRDE-DER3306.HRB12345` for a company in the commercial register of the Cologne Local Court. Under the BR, the registration number must be unique only where the register and legal system assign unique numbers.

## What the CA Verifies

Before issuing an OV or SV certificate, a CA must verify the following information under the BR:

- **Name and existence of the organization:** Through an authority in the organization’s country, LEI data, an on-site visit, or confirmation with a register extract. For a Swiss company, this means the commercial register and the UID register. If the company is listed there as deleted or inactive, the CA may not issue the certificate.
- **Registration number:** The CA confirms that the number in the `organizationIdentifier` belongs to the organization in the `O` field.
- **Authority:** Through a reliable communication channel, the CA verifies that the applicant is authorized to apply for certificates for the organization or to act as an `Enterprise RA`.
- **Mailbox or email domain:** The CA verifies control over the email addresses in the certificate. If the organization acts as an `Enterprise RA`, it must prove control over the entire email domain.
- **Individual (SV only):** The person’s name or pseudonym. The next section describes which evidence is permitted for this.

A CA may reuse the results of these verifications: organization and authority verification for 825 days, and domain-control verification for 398 days. A certificate of the `Multipurpose` and `Strict` generations is valid for no more than 825 days.

## How Individuals Are Verified for SV

The BR allow the following evidence for an individual’s identity:

| Evidence | Process |
|---|---|
| Government-issued ID | Original in person or in real time before a camera; the verifying person compares the ID photo with the individual and records the issuer, validity, and ID number |
| Chip in a passport or ID card | Verification of the issuer signature on the chip data, plus photo comparison |
| eID | Only notified eID systems under eIDAS with the “substantial” or “high” assurance level |
| Mobile driver’s license (mDL) | Real-time query under ISO/IEC 18013-5 and 18013-7, including verification of the issuer’s signature chain |
| Digital signature | The application is signed with an existing personal certificate from a recognized framework; the BR list of such frameworks is still empty |
| Confirmation by a notary’s office or law firm | Written confirmation from the applicant’s country |
| Records of the `Enterprise RA` | The company verifies its employees itself and may rely on existing internal data, such as human-resources records |

A CA must publish in its CPS which of this evidence it accepts. At issuance, verification of the individual may not be more than 825 days old. If the company verifies the individual itself as an `Enterprise RA`, the CA must bind it contractually and check it at least annually, unless the company itself is audited annually. In this case, the link to the individual is as reliable as the company’s internal process.

## SwissSign and DigiCert Compared

Both CAs offer the MV, SV, and OV types under their own product names. Neither offers an IV certificate. The BR define the structure of the `organizationIdentifier`, and it applies equally to both vendors.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (formerly Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | not offered | not offered |
| Purchase | Silver in the web shop or through Managed PKI; Gold and OV only through the company’s Managed PKI | all three through the company’s CertCentral account |
| Verification of the individual for SV | Original government-issued ID, in person or by an equivalent procedure; other evidence only with SwissSign approval | Confirmation by the company; DigiCert does not verify the individual |
| Validity of verifications | Information and evidence no more than 398 days old | Organization 825 days, domain 398 days |
| Generation for SV and OV | `Multipurpose`; an intermediate CA for SV in the `Strict` generation is already described in the CP/CPS | `Strict` as standard, `Multipurpose` selectable |
| Individual for SV | First and last name or pseudonym with the prefix `pseudo:` in the `CN` | First and last name or pseudonym; the `CN` may contain either the name or the email address |
| Additional uses for SV | None in the standard profile; client authentication, smart card logon, and EFS in the profile with authentication | Non-repudiation for `Strict`, plus data encryption and client authentication for `Multipurpose` |

DigiCert explicitly describes the SV product under the `Enterprise RA` model: the organization confirms that the individual in the certificate is one of its employees or representatives and retains the supporting documentation. DigiCert itself verifies only the organization. Anyone authorized to order through the company’s CertCentral account can therefore enter any name for an address in its own domains. For OV and SV, DigiCert always requires proof of domain control. Organization verification is valid for 825 days, and domain verification for 398 days. You can choose a one- or two-year term, or a custom expiration date of up to 824 days. DigiCert discontinued the former Premium, Email Security Plus, Digital Signature Plus, and Class 1 S/MIME products on July 10, 2025, replacing them with these products.

SwissSign issues Gold and OV certificates only through the company’s Managed PKI. The individual must nevertheless identify themselves using an original government-issued ID, in person or through an equivalent procedure under ETSI EN 319 411-1. Identification is performed by the registration authority processing the application or by a trained identification partner under contract: copying, scanning, or photographing the ID; comparing the photo with the individual; and confirming the process to SwissSign. Use of the company name must be approved by the executive management or board of directors registered in the commercial register, and the individuals applying for certificates in Managed PKI are also identified by ID. A company may use another procedure, such as one based on personnel data, only if SwissSign has reviewed and approved it as equivalent. Information and evidence are valid at SwissSign for no more than 398 days, after which SwissSign verifies them again.

For SwissSign, the OV certificate contains the organization name in the `CN`, as in the `O` field. The article [S/MIME at banks: legal situation in Switzerland and choosing between Silver and Gold](/blog/smime-silver-gold-schweiz) explains which level a bank should choose for which mailbox and what this means legally.

## Checking the Certificate Type

In Outlook and other email applications, you can open the certificate details of a signed email through the signature icon. In Windows, the fields appear on the “Details” tab under “Subject.” Windows has no display name for OID 2.5.4.97 and shows it as `OID.2.5.4.97`.

### Displaying the Holder with PowerShell

For an exported certificate (for example, from the certificate details of a signed email):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Get-PfxCertificate` | reads a certificate from a file, including a `.cer` file without a private key |
| `-FilePath .\zertifikat.cer` | path to the certificate file |
| `SubjectName.Format($true)` | outputs the holder’s fields one per line |

</details>

Output for an SV certificate (example):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

For an MV certificate, the output contains only `CN=` with the email address.

### Determining Type and Generation from the Policy OID

Every certificate under the BR contains an OID in the `Certificate Policies` extension following the pattern `2.23.140.1.5.<Typ>.<Generation>`:

| Position | Value | Meaning |
|---|---|---|
| Type | `1` | `Mailbox-validated` |
| Type | `2` | `Organization-validated` |
| Type | `3` | `Sponsor-validated` |
| Type | `4` | `Individual-validated` |
| Generation | `1` | `Legacy` |
| Generation | `2` | `Multipurpose` |
| Generation | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `$zert.Extensions` | extensions of the previously imported certificate |
| `'2.5.29.32'` | OID of the `Certificate Policies` extension |
| `Format($true)` | outputs the extension’s contents across multiple lines |

</details>

Output (abbreviated):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` indicates an SV certificate of the `Multipurpose` generation. Additional entries with OIDs of the CA itself are normal.

### Listing S/MIME Certificates in Your Own Certificate Store

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Cert:\CurrentUser\My` | personal certificate store of the signed-in user |
| `EnhancedKeyUsageList.ObjectId` | enhanced key usages of the certificate |
| `'1.3.6.1.5.5.7.3.4'` | OID for email protection (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | displays holder and expiration date |

</details>

### With OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `x509` | subcommand for certificates |
| `-in zertifikat.pem` | input file in PEM format; for a DER file, add `-inform der` |
| `-noout` | suppresses output of the encoded certificate |
| `-subject` | outputs the holder |
| `-nameopt multiline` | one field per line with fully spelled-out field names |

</details>

OpenSSL 3 recognizes the field name and displays it in full:

```text
subject=
    commonName                = Max Muster
    givenName                 = Max
    surname                   = Muster
    organizationName          = Muster AG
    organizationIdentifier    = NTRCH-CHE-123.456.789
    stateOrProvinceName       = ZH
    countryName               = CH
```

### Verifying the Registration Number

As the recipient of a signed email, you can verify the organization entry in four steps:

1. The email application reports the signature as valid, and the certificate comes from a CA trusted by the email application.
2. The policy OID begins with `2.23.140.1.5.2` or `2.23.140.1.5.3`, and the certificate contains the `O` and `organizationIdentifier` fields.
3. The UID from the `organizationIdentifier` belongs to the expected company in the UID register, and the company is listed as active there.
4. The sender address’s domain belongs to that company.

## Distinguishing It from ZertES

An organization entry does not make an S/MIME certificate a regulated certificate under ZertES. Only providers recognized under ZertES may issue regulated certificates, and the certificate must state that it is a regulated certificate (Art. 2 para. g and Art. 7 para. 2 lit. b ZertES).

There are substantive parallels. A regulated certificate for a UID entity must contain its UID (Art. 7 para. 2 lit. e ZertES), just as the `organizationIdentifier` contains it for OV and SV. A certificate for a natural person may additionally state that the person is authorized to represent a specific UID entity (Art. 7 para. 3 lit. b ZertES); structurally, this corresponds to an SV certificate.

| | S/MIME with organization entry | Regulated certificate under ZertES |
|---|---|---|
| Basis | S/MIME Baseline Requirements of the CA/Browser Forum | ZertES and implementing provisions |
| Issuer | publicly trusted CA | provider recognized under ZertES |
| Organization identifier | `organizationIdentifier`, for Swiss companies with the UID | UID, required for UID entities |
| Verification of the individual | for SV, depending on the CA, with ID or through confirmation by the organization as an `Enterprise RA` | generally personal appearance with proof of identity; for UID entities, a representative with proof of authority to represent (Art. 9 ZertES) |
| Key | according to CA requirements, including on a gateway | requirements under Art. 6 ZertES |

Under Art. 14 para. 2bis OR, only a qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature, and a qualified certificate may be issued only to a natural person (Art. 8 para. 1 ZertES). For an organization’s signature, ZertES provides for the regulated electronic seal: an advanced electronic signature created with a secure seal-creation device and based on a regulated certificate of a UID entity (Art. 2 para. d ZertES). An S/MIME-signed email is evidence whose value is freely assessed by the court (Art. 157 ZPO); a verified organization entry supports its attribution to the sending company.

**EU note:** In the EU, the eIDAS Regulation governs electronic signatures and seals. Under Art. 25 para. 2 eIDAS, only the qualified electronic signature is likewise equivalent to a handwritten signature.

## Recommendation

- **Functional mailboxes:** OV certificate (SwissSign Organization Validated, DigiCert Secure Email for Organization). It identifies the company without naming an individual.
- **Personal mailboxes used to sign communications to customers and partners:** SV certificate (SwissSign Gold, DigiCert Secure Email for Employee). It identifies the individual and the company.
- **Encryption only:** Cryptographically, all types are equivalent. An MV certificate is sufficient for encryption alone.
- **Plan for verifications:** Organization verification is valid for 825 days, and domain verification for 398 days. Following a change in company name or registered office, the information in the certificate no longer matches the register; plan for new certificates in that case.
- **Verify on receipt:** The policy OID and UID show whether a signature is assigned to a verified company. How thoroughly the individual was verified depends on the CA’s SV procedure: SwissSign uses ID verification, while DigiCert relies on the company’s confirmation.

## Sources

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Version 1.0.17 of November 20, 2026; certificate types and policy OIDs (Section 1.1), `Enterprise RA` (1.3.2), organization verification (3.2.3), individual verification (3.2.4), authority verification (3.2.6), reuse of verification results (4.2.1), validity periods (6.3.2), Subject fields (7.1.4.2), and registration schemes (Appendix A).

2.  [cabforum/smime on GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown version of the S/MIME Baseline Requirements with version history, including EUID from version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Product names Silver and Gold, example `NTRCH-CHE-109.357.012`, UID as registration number, and EUID for German companies (April 28, 2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): SwissSign’s combined policy document, version 1.0 of August 31, 2026, replacing, among other documents, the CPR S/MIME; individual identification (3.2.3), authority (3.2.5), validity of evidence (3.3.1), purchase channels (4.1.1), and certificate profiles for MV, SV, and OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Product overview with classification as MV, OV, and SV, and the end of the `Legacy` generation and former S/MIME products on July 10, 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organization and domain verification, confirmation of employees by the organization without individual verification by DigiCert, `Strict` and `Multipurpose` profiles, and validity periods.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (terms, regulated seal), Art. 6 (keys), Art. 7 (contents of regulated certificates, UID), Art. 8 (qualified certificates), and Art. 9 (identity verification).

8.  [Swiss Code of Obligations (OR)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 para. 2bis on the equivalence of the qualified electronic signature to a handwritten signature.

9.  [UID Register](https://www.uid.admin.ch): Lookup of a UID with name, registered office, and status, maintained by the Federal Statistical Office.

10.  [Zefix](https://www.zefix.ch): Central company index of the cantonal commercial registers.

11.  [Swiss Civil Procedure Code (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 on the free assessment of evidence.

12.  [Regulation (EU) No. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 para. 2 on the equivalence of the qualified electronic signature to a handwritten signature.
