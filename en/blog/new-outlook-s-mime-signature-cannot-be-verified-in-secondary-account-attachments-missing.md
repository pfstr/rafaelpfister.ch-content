---
title: "New Outlook: S/MIME signature cannot be verified in secondary account, attachments missing"
navTitle: "S/MIME in secondary account"
description: "The new Outlook reports that the S/MIME signature cannot be verified in a secondary account for a shared mailbox and does not display attachments. This article explains the difference between Clear Signing and Opaque Signing, why attachments disappear from opaque-signed messages, why the new Outlook processes S/MIME only in the primary account, and what workarounds are available, including unpacking smime.p7m with PowerShell or OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min read"
themen:
  - outlook
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "outlook"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
slug: "new-outlook-s-mime-signature-cannot-be-verified-in-secondary-account-attachments-missing"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/en/blog/new-outlook-s-mime-signature-cannot-be-verified-in-secondary-account-attachments-missing
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:08:55.760Z
translationReview: automatic
---

# New Outlook: S/MIME signature cannot be verified in secondary account, attachments missing

When opening a digitally signed email in a shared mailbox in the new Outlook for Windows, a red banner appears: "The S/MIME signature cannot be verified when viewing in a secondary account." The email itself is displayed, but its attachments are not, even though the sender included them. Colleagues who use the same mailbox as their primary account can see the attachments without any issues.

Two things are behind this, and they compound each other: the new Outlook processes S/MIME only in the primary account, and the sender signed the email opaquely. With this signing format, the complete content, including attachments, is contained in a single cryptographic container. If the client cannot open the container, the attachments remain invisible. Both issues can be resolved separately.

## What the message means

In the new Outlook, "secondary account" means any mailbox other than the account you signed in with. This applies to shared mailboxes that are displayed through Full Access and automapping, as well as mailboxes you added through "Add shared mailbox," and delegated mailboxes. S/MIME processing in the new Outlook is firmly tied to the primary account. When a signed message arrives in another account, the client does not verify the signature and displays the message instead.

This does not indicate anything about the validity of the signature and is not a certificate issue on the sender's end. The same email can be verified and opened in the primary account or in classic Outlook.

## Clear Signing and Opaque Signing

The S/MIME standard (RFC 8551) defines two formats for signed messages. Both provide the same signature, but package the message differently.

| | Clear Signing | Opaque Signing |
|---|---|---|
| MIME type | `multipart/signed` with `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` with `smime-type=signed-data` |
| Structure | Two parts: the readable message body including attachments, plus the detached signature | One part: message body, attachments, and signature together in a CMS SignedData container, Base64-encoded |
| Attachment seen by a client without S/MIME | `smime.p7s` (only the signature, a few KB) | `smime.p7m` (the entire message) |
| Readable without S/MIME support | Yes, text and attachments are displayed normally | No, the client sees only the container |
| Sensitivity during transport | The signature becomes invalid if a mail server or gateway changes line breaks, encoding, or whitespace | The container protects the content from such changes |
| RFC 8551 section | 3.5.3 | 3.5.2 |

You can identify the two formats in the message source by the `Content-Type` header. A clear-signed email begins like this:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

An opaque-signed email begins like this:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

The difference fully explains the behavior in the new Outlook. With a clear-signed email, the client displays text and attachments even when it does not verify the signature; only the signature status is missing. With an opaque-signed email, the client must first unpack the container through S/MIME processing to access the text and attachments. If it refuses because the message is in a secondary account, the container remains closed. The fact that the text is still readable is due to Exchange Online: the service renders the text portion server-side, but not the attachments from the container.

Neither format encrypts anything. Even the opaque variant is only Base64-encoded and can be read by anyone who gets hold of the message. Microsoft explicitly points this out in its Exchange Online documentation.

## Which format the sender chooses

In classic Outlook, the "Send clear text signed messages" option (File > Options > Trust Center > Email Security) controls the format. It is enabled by default, so Outlook signs emails as clear-signed. Anyone who disables the option sends opaque-signed messages. The new Outlook and Outlook on the web do not offer this setting.

Mail gateways that sign centrally have their own setting for the signature format. Some products sign opaquely by default for robustness, because the signature then remains valid even after rewrapping by downstream systems. If you regularly receive emails with missing attachments from a particular sender, it is worth checking that sender's gateway configuration.

## Why the new Outlook processes S/MIME only in the primary account

Microsoft documents the limitation but does not state a technical reason. The reason follows from the client's architecture.

At its core, the new Outlook is the Outlook on the web client in a native shell. In a browser, JavaScript cannot access the Windows certificate store. This is why Outlook on the web required a separate S/MIME browser extension for years. The new Outlook replaces this extension with a built-in bridge between the web interface and Windows cryptography. This bridge is initialized when signing in to the primary account and receives that mailbox's session, certificates, and S/MIME settings from Settings > Mail > S/MIME.

Shared mailboxes and secondary accounts use different data paths. Secondary accounts have their own sessions, while shared mailboxes are loaded through delegation from the primary account. So far, the bridge has not been connected for these paths. This also applies to merely verifying a signature, even though no private key would be required: parsing and unpacking the PKCS#7 structure uses the same component.

The problem does not occur in classic Outlook because S/MIME processing takes place in the MAPI stack on a per-message basis, regardless of which store the message comes from.

Microsoft is adding the missing integration: Roadmap item 565861 extends S/MIME in the new Outlook to shared and delegated mailboxes attached to the primary account. General availability is announced for July 2026, with a phased rollout. If you still see the message, the change has not yet reached your tenant or client version. The item does not cover separately added secondary accounts with their own sign-in.

## Workarounds

Which option fits depends on how the mailbox is connected and whether you need to verify the signature or just access the attachments.

| Option | Requirement | Result |
|---|---|---|
| Open the email in the primary account | You have the mailbox itself as your primary account, or the email was forwarded to you | Signature verification and attachments |
| Add the mailbox as a separate account in the new Outlook (Settings > Accounts > Add account) | The mailbox has its own sign-in credentials; not possible for shared mailboxes without a password | Signature verification and attachments once you switch to that account |
| Classic Outlook | Still installed or available by switching back using the "New Outlook" toggle; add the mailbox there as a separate account (File > Account Settings > New) | Signature verification and attachments in every store |
| Outlook on the web | Open the mailbox directly (`outlook.office.com/mail/<adresse>`), with the S/MIME extension for Edge or Chrome installed | Signature verification and attachments |
| Ask the sender for Clear Signing | The sender uses classic Outlook or a gateway with a selectable format | Attachments visible, but no signature status in the secondary account |
| Unpack the container manually | Save the email as `.eml` or save `smime.p7m` | Attachments without signature verification |

## Unpacking the container manually

For an isolated case, simply open the container outside Outlook. The signature is cryptographically verified in the process, but the certificate trust chain is not. Save the message as `.eml` or save the `smime.p7m` attachment to a folder.

On Windows, PowerShell is sufficient. The .NET Framework includes the `SignedCms` class, which reads the PKCS#7 container:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Options explained</summary>

| Statement | Effect |
|---|---|
| `Add-Type -AssemblyName System.Security` | Loads the assembly containing the PKCS classes (required in Windows PowerShell 5.1; already loaded in PowerShell 7) |
| `[IO.File]::ReadAllBytes(...)` | Reads the binary DER container; the `smime.p7m` saved from Outlook is already decoded |
| `$cms.Decode($bytes)` | Parses the CMS SignedData structure |
| `$cms.CheckSignature($true)` | Verifies only the signature over the content (`$true` = verifySignatureOnly); `$false` would additionally verify the certificate chain. If the signature is invalid, the command stops with an exception |
| `$cms.ContentInfo.Content` | The signed content: a complete MIME message with text and attachments |
| `[IO.File]::WriteAllBytes(...)` | Writes this MIME message as `.eml`, which you can open in Outlook or Thunderbird |

</details>

On Linux, macOS, or with Git for Windows, OpenSSL is available. If the entire email is available as `.eml`, OpenSSL also handles Base64 decoding:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `cms` | OpenSSL's CMS tool; `smime` works equivalently, while `cms` is the newer interface |
| `-verify` | Verifies the signature and outputs the signed content |
| `-noverify` | Skips certificate chain verification; the signature itself is still verified |
| `-in nachricht.eml` | The complete email in S/MIME format (Base64 with MIME headers); for a saved `smime.p7m`, also use `-inform DER` |
| `-out inhalt.eml` | The unpacked content as a MIME message |

</details>

The file `inhalt.eml` contains the original message text and all attachments as normal MIME parts. Double-clicking it opens it in Outlook, where you can save the attachments as usual.

## Sources

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): The real-world case with the same message in a shared mailbox; the answer confirms the behavior is known and provides no workaround in the new Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Sections 3.5.2 (application/pkcs7-mime with SignedData) and 3.5.3 (multipart/signed), including statements on readability without S/MIME and robustness during transport.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): The "Send clear text signed messages" option in classic Outlook, enabled by default; not available in the new Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): S/MIME settings in the new Outlook under Settings > Mail > S/MIME; certificates must be installed manually or through policy.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Notes that opaque-signed messages are only Base64-encoded and are not confidential.

6.  [Microsoft 365 Roadmap, item 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME for shared and delegated mailboxes in the new Outlook for Windows, announced for July 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Which account types the new Outlook supports and how shared mailboxes are connected.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature, and ContentInfo for unpacking the container with PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Options `-verify`, `-noverify`, `-inform` and `-out`.
