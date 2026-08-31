---
title: "compauth in Microsoft 365: Composite Authentication and All Reason Codes"
navTitle: "compauth Codes"
description: "Microsoft 365 supplements SPF, DKIM, and DMARC with its own evaluation: compauth. What Composite Authentication checks, what pass, softpass, fail, and none mean, and what causes each reason code, from 000 to 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min read"
themen:
  - microsoft-365-exchange
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-hybrid-header-intern-extern
  - dns-records-e-mail-stolpersteine
slug: "compauth-in-microsoft-365-composite-authentication-and-all-reason-codes"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/en/blog/compauth-in-microsoft-365-composite-authentication-and-all-reason-codes
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:20:30.585Z
translationReview: automatic
---

# compauth in Microsoft 365: Composite Authentication and All Reason Codes

In the `Authentication-Results` header of an email received in Microsoft 365, alongside the standard results for SPF, DKIM, and DMARC, there is a Microsoft-specific field:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` stands for Composite Authentication: Microsoft 365 combines the results of SPF, DKIM, and DMARC with other message signals into an overall assessment of whether the visible From address is credible. The assessment is based on the From domain—that is, the address recipients see in their email client. This closes the gap that arises when a sender domain has published no or incomplete authentication records: even without a DMARC policy, Microsoft implicitly checks whether the email matches the claimed domain.

## The Four Results

- `compauth=pass`: The message passed explicit (DMARC) or implicit authentication.
- `compauth=softpass`: The implicit check passed with lower confidence.
- `compauth=fail`: The message failed the explicit or implicit check.
- `compauth=none`: No Composite Authentication check took place, or it was skipped.

A `compauth=fail` does not automatically result in quarantine or the Junk Email folder. It is an input signal for the filtering decision; `CAT` and other fields in `X-Forefront-Antispam-Report` determine the actual handling. Conversely, anyone wanting to know why compauth made its decision needs the `reason` code directly after the result.

## Reason Codes at a Glance

The three-digit code identifies the rule that led to the result. The first digit indicates the group: 0xx and 6xx are failures, 1xx and 7xx are passed checks, 2xx is softpass, and 3xx, 4xx, and 9xx mean no check or a skipped check.

| Code | Meaning |
|---|---|
| `000` | Explicit failure: DMARC fail with a `p=quarantine` or `p=reject` policy. |
| `001` | Implicit failure: The domain publishes no authentication records or only weak ones (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | The organization has explicitly prohibited sending spoofed email for this sender/domain pair (manually maintained entry). |
| `010` | DMARC fail with `p=reject`/`p=quarantine`, and the sending domain is one of its own Accepted Domains (spoofing of the organization's own domain). |
| `100` | SPF or DKIM passed, and the MAIL FROM and From domains are aligned. |
| `101` | The message is DKIM-signed by the From domain. |
| `102` | MAIL FROM and From domains are aligned, and SPF passed. |
| `103` / `104` | The From domain matches the PTR record (reverse lookup) of the submitting IP address. |
| `108` | DKIM fail due to a body modification at previous legitimate hops, such as in an on-premises environment. |
| `109` | The domain has no DMARC record, but the check would have passed. |
| `111` | Despite a DMARC temporary or permanent error, the SPF or DKIM domain is aligned with the From domain. |
| `112` | A DNS timeout prevented retrieval of the DMARC record. |
| `115` | The email originates from a Microsoft 365 organization where the From domain is configured as an Accepted Domain. |
| `116` | The MX record of the From domain matches the PTR record of the submitting IP. |
| `130` | An ARC sealer configured as trusted overrode the DMARC fail. |
| `201` / `202` | Softpass: The From domain matches the PTR record or its subnet, respectively. |
| `3xx` / `4xx` / `9xx` | No Composite Authentication check was performed or it was skipped. |
| `501` / `502` | DMARC was not enforced because this is a valid NDR. |
| `601` | Implicit failure: The sending domain is one of its own Accepted Domains (self-spoofing, often with Direct Send). |
| `701`–`704` | DMARC was not enforced because the organization demonstrably receives legitimate email from this infrastructure. |
| `905` | DMARC was not enforced due to complex routing, such as Internet email via on-premises Exchange or a third-party service before Microsoft 365. |

## The Most Common Cases in Practice

**`compauth=fail reason=001`** is the standard case for domains without authentication or with weak authentication. The fix lies with the sender: publish SPF with `-all`, DKIM signing, and a DMARC policy. As long as the records are missing, deliverability depends on reputation signals.

**`compauth=fail reason=601`** appears when email using the organization's own domain as the sender arrives from outside, classically with Direct Send: multifunction devices, applications, or service providers deliver directly to the MX without an authenticated connector. The fix is a properly configured inbound connector or adding the source to the organization's own SPF.

**`compauth=fail reason=000` or `010`** means that DMARC was applied as intended. If `action=oreject` appears alongside it, Microsoft 365 translated the sender's reject policy into quarantine delivery. There is nothing to fix here unless the sender is legitimate and its authentication is broken.

**`reason=108`** and **`reason=130`** concern forwarding and gateway scenarios: an intermediate hop modified the email, or a trusted ARC sealer preserved the original validation results. Anyone operating a gateway in front of Microsoft 365 should mark its ARC sealing as trusted in the anti-spam configuration; otherwise, legitimate email may continue to fail DMARC.

## Reading compauth in the Header

In practice, `compauth` rarely stands alone: only its interaction with the individual SPF, DKIM, and DMARC results, the alignment of the domains involved, and the `Received` chain provides the complete picture. The [Mail Header Analyzer](/tools/header-analyzer) on this website decodes `compauth` including its reason code directly in the browser and displays the associated domains (From, Envelope From, `d=`) side by side for alignment assessment; the pasted header never leaves the browser.

## Sources

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Official reference for the Authentication-Results fields and the complete compauth reason code table.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Guidance for handling authentication failures from a SecOps perspective.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Configuring trusted ARC sealers for gateway and forwarding scenarios (reason code 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Distinguishing between the compauth signal and the actual filtering decision.
