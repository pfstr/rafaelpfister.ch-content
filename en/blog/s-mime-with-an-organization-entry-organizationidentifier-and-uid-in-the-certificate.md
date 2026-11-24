---
title: "S/MIME with an Organization Entry: organizationIdentifier and UID in the Certificate"
navTitle: "S/MIME with Organization"
description: "What an S/MIME certificate with an organization entry contains, how the UID is included in the certificate as an organizationIdentifier, and what the certification authority verifies for this purpose. Also includes a comparison of SwissSign and DigiCert products, commands for checking the certificate type, and the distinction from ZertES."
date: "2026-12-15"
kategorie: "Email Encryption"
timeToRead: "11 min read"
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
url: https://rafaelpfister.ch/en/blog/s-mime-with-an-organization-entry-organizationidentifier-and-uid-in-the-certificate
translationSourceHash: f628a8f86635449d2a9e43f78f1a960ce5c8bc20af519eac110b77b2dad385a9
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:51:09.396Z
translationReview: required
---

# S/MIME with an Organization Entry: organizationIdentifier and UID in the Certificate

Certification authorities (CAs) distinguish between S/MIME certificates with and without an organization entry. In addition to the email address, a certificate with an organization entry identifies the company to which the mailbox belongs: by its verified name and a registry number. For Swiss companies, this registry number is the UID, i.e., the CHE number from the commercial register.

In brief: An organization entry is available only for the `Organization-validated` and `Sponsor-validated` certificate types under the S/MIME Baseline Requirements. It consists of the company name in the `O` field and the `organizationIdentifier` field (OID 2.5.4.97), for a Swiss company, for example, `NTRCH-CHE-123.456.789`. The CA verifies the company in an official registry before entering this information in the certificate. By contrast, a mailbox-validated certificate proves only access to a mailbox. With SwissSign, this corresponds to choosing between Silver and Gold; with DigiCert, to choosing between “Secure Email for Individual Mailbox” and the “for Employee” and “for Organization” products. An organization entry does not create a qualified signature under ZertES.

**Switzerland focus:** The legal classification follows Swiss law (ZertES, CO). Where EU rules differ, a labeled note is provided.

## The Four Certificate Types

Since September 1, 2023, the `S/MIME Baseline Requirements` (BR) of the CA/Browser Forum have applied to publicly trusted S/MIME certificates. They distinguish four certificate types based on who is listed as the holder (`Subject`) in the certificate:

| Type | Holder in the certificate | Organization entry | What the CA verifies |
|---|---|---|---|
| `Mailbox-validated` (MV) | only the email address | not permitted | Control of the mailbox or email domain |
| `Organization-validated` (OV) | an organization, no person | required | Organization and mailbox or email domain |
| `Sponsor-validated` (SV) | a person affiliated with an organization | required | As for OV, plus the person; the CA or the organization itself verifies that person |
| `Individual-validated` (IV) | a person without an organization | not permitted | The person’s identity and mailbox |

“S/MIME with an organization entry” refers to the OV and SV types. For both, the BR require the `organizationName` and `organizationIdentifier` fields in all three generations: `Legacy`, `Multipurpose`, and `Strict`. The generations differ in the permitted Subject fields, extensions, and purposes. The BR deprecated the `Legacy` generation as of July 15, 2025; SwissSign and DigiCert have not issued it since then.

OV certificates are suitable for functional mailboxes such as `info@` or `buchhaltung@`, which are not assigned to an individual. SV certificates are intended for employees’ personal mailboxes and additionally include the person’s name or a pseudonym.

## Structure of the organizationIdentifier

The `organizationIdentifier` field (OID 2.5.4.97) contains an organization registry number. Certificates under eIDAS and ZertES use the same field. For S/MIME certificates, the BR prescribe the following structure:

| Part | Example | Meaning |
|---|---|---|
| Registration scheme, 3 characters | `NTR` | `National Trade Register`, the commercial register |
| Country code under ISO 3166-1 | `CH` | Country in which the registry is maintained |
| Hyphen | `-` | Separator |
| Registry number | `CHE-123.456.789` | Organization’s number in the registry |

Combined, this results in `NTRCH-CHE-123.456.789`. Because the UID itself contains a hyphen, the BR specify that the scheme and country code must never contain a hyphen. The first hyphen from the left therefore separates the scheme from the registry number; all subsequent hyphens belong to the number. If a registry issues numbers only at the level of a state or province, the country code is followed by a `+` with the region code, such as `NTRUS+CA-12345678` for California.

The BR recognize these registration schemes:

| Scheme | Origin of the number | Example from the BR |
|---|---|---|
| `NTR` | Commercial or business register | `NTRGB-12345678` |
| `VAT` | Tax authority (VAT number) | `VATEL-123456789` |
| `PSD` | Authorization number of a payment service provider under EU Directive 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` under ISO 17442, country code always `XG` | `LEIXG-` followed by the LEI |
| `GOV` | Government entities without a number from one of the other schemes | `GOVUS` |
| `INT` | International organizations | `INTXG` |

For Swiss companies, `NTR` with the UID is the standard case. The UID is assigned by the Federal Statistical Office. Commercial registers use it as the company number; it also serves as the VAT number, and it remains unchanged when a company moves its registered office to another canton. A VAT number, by contrast, may cease to apply when tax liability ends, and not every organization has one. SwissSign adopts the UID with the periods, as listed in the UID register, and gives `NTRCH-CHE-109.357.012` as an example for SwissSign AG.

**EU note:** In Germany, the local courts assign commercial register numbers, and the same number may occur at multiple courts. Since version 1.0.9 of May 2025, the BR have therefore allowed the `European Unique Identifier` (EUID) under the `NTR` scheme and recommend it for German companies, for example `NTRDE-DER3306.HRB12345` for a company in the commercial register of the Cologne Local Court. Under the BR, the registry number must be unique only where the registry and legal system issue unique numbers.

## What the CA Verifies

Before issuing an OV or SV certificate, a CA must verify the following information under the BR:

- **Name and existence of the organization:** Through a government authority in the organization’s country, LEI data, an on-site visit, or confirmation with a registry extract. For a Swiss company, this means the commercial register and the UID register. If the company is listed there as deleted or inactive, the CA must not issue the certificate.
- **Registry number:** The CA confirms that the number in the `organizationIdentifier` belongs to the organization in the `O` field.
- **Authorization:** Through a reliable communication channel, the CA verifies that the applicant is authorized to apply for certificates for the organization or act as an `Enterprise RA`.
- **Mailbox or email domain:** The CA verifies control of the email addresses in the certificate. If the organization acts as an `Enterprise RA`, it must demonstrate control over the entire email domain.
- **Person (SV only):** The BR allow the organization, as an `Enterprise RA`, to confirm its employees itself. The CA then verifies the organization rather than each individual person.

A CA may reuse the results of these verifications: verification of the organization and authorization for 825 days, and verification of domain control for 398 days. A certificate in the `Multipurpose` and `Strict` generations is valid for no more than 825 days.

## SwissSign and DigiCert Compared

Both CAs offer the MV, SV, and OV types under their own product names. Neither offers an IV certificate. The BR define the structure of the `organizationIdentifier`, and it is the same for both vendors.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (formerly Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | not offered | not offered |
| Generation for SV and OV | `Multipurpose` according to certificate profiles | `Strict` as standard, `Multipurpose` selectable |
| Person in SV | First and last name or pseudonym with the `pseudo:` prefix in the `CN` | First and last name or pseudonym; the `CN` can contain either the name or the email address |
| Additional purposes in SV | In the profile with authentication: client authentication, smart card logon, EFS | For `Strict`, non-repudiation; for `Multipurpose`, additionally data encryption and client authentication |

DigiCert explicitly describes the SV product under the `Enterprise RA` model: the organization confirms that the person in the certificate is one of its employees or representatives and retains the supporting documentation. DigiCert itself verifies only the organization. For OV and SV, DigiCert always requires proof of domain control. Organization verification is valid for 825 days, and domain verification for 398 days. Available validity periods are one or two years, or a custom expiration date of up to 824 days. DigiCert discontinued its former Premium, Email Security Plus, Digital Signature Plus, and Class 1 S/MIME products on July 10, 2025, replacing them with these products.

For SwissSign, the OV certificate contains the organization’s name in the `CN`, as in the `O` field. The article [S/MIME at Banks: Legal Situation in Switzerland and Choosing Between Silver and Gold](/blog/smime-silver-gold-schweiz) describes which level a bank should choose for which mailbox and what this means legally.

## Checking the Certificate Type

In Outlook and other email clients, you can open the certificate details of a signed email through the signature icon. In Windows, the fields appear on the “Details” tab under “Subject.” Windows has no display name for OID 2.5.4.97 and displays it as `OID.2.5.4.97`.

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

### Determining Type and Generation Using the Policy OID

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
| `$zert.Extensions` | extensions of the previously loaded certificate |
| `'2.5.29.32'` | OID of the `Certificate Policies` extension |
| `Format($true)` | outputs the extension’s contents across multiple lines |

</details>

Output (abbreviated):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` represents an SV certificate of the `Multipurpose` generation. Additional entries with the CA’s own OIDs are normal.

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
| `EnhancedKeyUsageList.ObjectId` | extended key usages of the certificate |
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
| `x509` | certificate subcommand |
| `-in zertifikat.pem` | input file in PEM format; for a DER file, additionally use `-inform der` |
| `-noout` | suppresses output of the encoded certificate |
| `-subject` | outputs the holder |
| `-nameopt multiline` | one field per line with full field names |

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

### Verifying the Registry Number

As the recipient of a signed email, you can verify the organization entry in four steps:

1. The email client reports the signature as valid, and the certificate comes from a CA trusted by the email client.
2. The policy OID begins with `2.23.140.1.5.2` or `2.23.140.1.5.3`, and the certificate contains the `O` and `organizationIdentifier` fields.
3. The UID from the `organizationIdentifier` belongs to the expected company in the UID register, and the company is active there.
4. The sender address’s domain belongs to that company.

## Distinction from ZertES

The organization entry does not make an S/MIME certificate a regulated certificate under ZertES. Only providers recognized under ZertES issue regulated certificates, and the certificate must indicate that it is a regulated certificate (Art. 2 para. g and Art. 7 para. 2 lit. b ZertES).

There are substantive parallels. A regulated certificate for a UID entity must contain its UID (Art. 7 para. 2 lit. e ZertES), just as the `organizationIdentifier` contains it for OV and SV. A certificate for a natural person may also contain an indication that the person is authorized to represent a specific UID entity (Art. 7 para. 3 lit. b ZertES); in structure, this corresponds to an SV certificate.

| | S/MIME with organization entry | Regulated certificate under ZertES |
|---|---|---|
| Basis | S/MIME Baseline Requirements of the CA/Browser Forum | ZertES and implementing provisions |
| Issuer | publicly trusted CA | provider recognized under ZertES |
| Organization identifier | `organizationIdentifier`, for Swiss companies with the UID | UID, mandatory for UID entities |
| Verification of the person | For SV, by the CA or the organization as an `Enterprise RA` | generally personal appearance with proof of identity; for UID entities, a representative with proof of authority to represent (Art. 9 ZertES) |
| Key | under the CA’s requirements, including on a gateway | requirements under Art. 6 ZertES |

Under Art. 14 para. 2bis CO, only a qualified electronic signature with a qualified timestamp is equivalent to a handwritten signature, and a qualified certificate may be issued only to a natural person (Art. 8 para. 1 ZertES). For an organization’s signature, ZertES provides for the regulated electronic seal: an advanced electronic signature created using a secure seal-creation device and based on a regulated certificate of a UID entity (Art. 2 para. d ZertES). An S/MIME-signed email is evidence whose weight is freely assessed by the court (Art. 157 CPC); a verified organization entry supports its attribution to the sending company.

**EU note:** In the EU, the eIDAS Regulation governs electronic signatures and seals. Under Art. 25 para. 2 eIDAS, only the qualified electronic signature is likewise equivalent to a handwritten signature.

## Recommendation

- **Functional mailboxes:** OV certificate (SwissSign Organization Validated, DigiCert Secure Email for Organization). It identifies the company without naming an individual.
- **Personal mailboxes used to sign communications with customers and partners:** SV certificate (SwissSign Gold, DigiCert Secure Email for Employee). It identifies the person and company.
- **Encryption only:** Cryptographically, all types are equivalent. An MV certificate is sufficient for encryption alone.
- **Plan for verification:** Organization verification is valid for 825 days, and domain verification for 398 days. After a change of company name or registered office, the information in the certificate no longer matches the registry; plan for new certificates in that case.
- **Verify on receipt:** The policy OID and UID show whether a signature is actually assigned to a verified company.

## Sources

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): Version 1.0.17 of November 20, 2026; certificate types and policy OIDs (Section 1.1), organization verification (3.2.3), authorization verification (3.2.6), reuse of verification results (4.2.1), validity periods (6.3.2), Subject fields (7.1.4.2), and registration schemes (Appendix A).

2.  [cabforum/smime on GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): Markdown version of the S/MIME Baseline Requirements with version history, including EUID from version 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): Silver and Gold product names, example `NTRCH-CHE-109.357.012`, UID as registry number, and EUID for German companies (April 28, 2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): SwissSign certificate profiles for MV, SV, and OV with Subject fields, purposes, and policy OIDs, version 14 of September 18, 2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): Product overview mapping to MV, OV, and SV; end of the `Legacy` generation and the former S/MIME products on July 10, 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): Organization and domain verification, confirmation of employees by the organization, `Strict` and `Multipurpose` profiles, and validity periods.

7.  [Federal Act on Electronic Signatures (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): Art. 2 (terms, regulated seal), Art. 6 (keys), Art. 7 (content of regulated certificates, UID), Art. 8 (qualified certificates), and Art. 9 (identity verification).

8.  [Swiss Code of Obligations (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): Art. 14 para. 2bis on equivalence of the qualified electronic signature to a handwritten signature.

9.  [UID Register](https://www.uid.admin.ch): Lookup of a UID with name, registered office, and status, maintained by the Federal Statistical Office.

10.  [Zefix](https://www.zefix.ch): Central company index of the cantonal commercial registers.

11.  [Swiss Civil Procedure Code (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): Art. 157 on the free assessment of evidence.

12.  [Regulation (EU) No. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): Art. 25 para. 2 on equivalence of the qualified electronic signature to a handwritten signature.
