---
title: "Outlook.com Rejects Email Without a DMARC Record: Error AS9031519"
navTitle: "Outlook.com: AS9031519"
description: "Since November 5, 2026, Outlook.com, Hotmail, and Live have been rejecting messages from domains without a DMARC record with 550 5.7.1 and the AS9031519 suffix, even at low sending volumes. What Microsoft documents, what operators have observed, how to check your own domain, and what the mail security index shows for Swiss municipalities."
date: "2026-11-10"
kategorie: "SMTP and Mail Flow"
timeToRead: "8 min read"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - dns-records-e-mail-stolpersteine
  - microsoft-365-compauth-reason-codes
slug: "outlook-com-rejects-email-without-a-dmarc-record-error-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/en/blog/outlook-com-rejects-email-without-a-dmarc-record-error-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:19:27.797Z
translationReview: required
---

# Outlook.com Rejects Email Without a DMARC Record: Error AS9031519

Since November 5, 2026, messages sent to mailboxes at Outlook.com, Hotmail, Live, and MSN have been bouncing with a new error when the sender domain has not published a DMARC record:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Information current as of November 10, 2026.

## What Microsoft documents

Outlook.com's postmaster policies still contain only the rule from May 5, 2025: Domains that send more than 5,000 messages per day to Outlook.com accounts must pass SPF and DKIM and publish a DMARC record with at least `p=none`, aligned through SPF or DKIM. Messages that do not meet these requirements first end up in the Junk Email folder and may later be rejected with the following error:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

For all other senders, the same page merely makes a recommendation. The suffix `AS9031519` and the error text `Message lacks SPF/DKIM/DMARC Auth Records` appear neither in the postmaster policies nor in the support article on 5.7.515.

Microsoft has commented on the current situation only in the Microsoft Q&A forums. On November 9, a moderator identified as “Microsoft External Staff” wrote that an ongoing issue in the Exchange service was affecting delivery to Outlook.com, Hotmail.com, and Live.com. It was classified as a service degradation, and Microsoft was investigating the cause. The post does not provide an incident number, a cause, or a date for a fix. There is no blog post, postmaster-policy entry, or public service-health notice.

## What operators have observed

The most reliable data comes from mail providers that analyzed their logs and published their findings on the mailop mailing list and in their own blog posts (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Their observations align:

- **Trigger:** The sender domain has no DMARC record. SPF and DKIM were valid and aligned in the documented cases. Although the error text mentions all three mechanisms, rejection is due to the missing DMARC record.
- **Volume:** Domains with only a few messages per week are also affected, far below the threshold of 5,000 per day.
- **Recipients:** Only consumer mailboxes (outlook.com, hotmail.com, live.com, msn.com, and country variants such as hotmail.de or outlook.be). According to the available reports, Exchange Online tenants as recipients do not reject with this error.
- **Intermittent:** Only some messages are rejected. Resending the same message may succeed.
- **Resolution:** A record with `v=DMARC1; p=none` was sufficient in the documented cases. After publication, it took anywhere from a few minutes to about an hour and a half before Outlook.com accepted messages again.

Two analyses narrow down the trigger further: At one.com, only domains whose query for `_dmarc.<domain>` ended with `NXDOMAIN` were rejected. Domains for which the name existed but no TXT record was present (`NOERROR` without an answer) were not affected there. MXroute found 367 rejected sender domains in one log, 286 of them with `NXDOMAIN` for `_dmarc`. This narrowing is based on data from individual operators and has not been conclusively clarified.

Since midday on November 9, some affected users have reported that the rejections have stopped. Microsoft has not confirmed this.

## Special case: Microsoft 365 as the sender

Microsoft 365 tenants are disproportionately affected as senders. Microsoft 365 does not create a DMARC record for a custom domain, and until DKIM is enabled for the domain, Exchange Online signs with the tenant’s `onmicrosoft.com` domain. The non-delivery report additionally wraps the rejection in a second status code that refers to SPF:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

In these cases, the SPF record is generally correct. What matters is the inner error after the arrow.

## Checking your own domain

The TXT record queried is under `_dmarc.<domain>`:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | `dig` outputs only the record contents |
| `TXT` or `-Type TXT` | Query record type |
| `_dmarc.example.com` or `-Name` | Queried name: the `_dmarc` label before the sender domain |

</details>

Empty output from `dig +short` or the error `DNS name does not exist` from `Resolve-DnsName` means that no DMARC record exists. For subdomains: If `news.example.com` sends mail and has no record of its own, RFC 7489 uses the record of the organizational domain `example.com`.

The [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check) on this site checks DMARC together with SPF, DKIM, and alignment in one pass.

## Immediate action and what comes next

If the record is missing, this entry restores delivery to Outlook.com:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` does not change how other recipients handle your email and can therefore be published without preparation. The `rua` tag is not required for acceptance by Outlook.com. Without `rua`, however, you will not receive aggregate reports, and without those reports there is no basis for any further step. The address must reach a mailbox or analysis service that processes the XML reports.

`p=none` does not protect the domain against abuse. The record merely meets the acceptance requirement. For effective protection, follow these steps:

1. Identify all systems that send using the domain (mail servers, newsletter service, ERP, web store, scanners).
2. Keep SPF complete and valid for these systems, with no more than ten DNS lookups.
3. Enable DKIM with your own domain, in Microsoft 365 for each domain in the Defender portal.
4. Analyze aggregate reports for several weeks and correct unaligned sources.
5. Raise the policy to `p=quarantine` and then to `p=reject`.

The [Guide for DNS admins](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) describes common errors when publishing these records (duplicate SPF records, DKIM keys that are too long, automatically appended zone names).

## The impact in Switzerland: 152 municipalities in five days

The [Swiss Mail Security Index](https://rafaelpfister.ch/open-data/mail-security-index) checks the DNS records of 3,364 Swiss organizations every night, including 2,078 municipalities. The [change log](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) shows a wave of newly published DMARC records since November 6. Each date is that of the nightly comparison.

| Comparison | Organizations newly with a DMARC record | Of these, municipalities |
|---|---|---|
| October 19 through November 5 | at most 5 per day | at most 4 per day |
| November 6 | 4 | 4 |
| November 7 | 22 | 21 |
| November 8 | 1 | 1 |
| November 9 | 2 | 2 |
| November 10 | 140 | 124 |

In five days, 169 organizations published a DMARC record for the first time, 152 of them municipalities. The number of organizations without DMARC in the index fell from 817 to 650. The timing coincides with the Outlook.com rejections. I did not ask the municipalities themselves about the reasons.

The records have the following characteristics:

- **Policy:** 139 of the 152 municipalities set `p=none`, five set `p=quarantine`, and three set `p=reject`. For five, the record consists only of `v=DMARC1` without a `p=` tag and is therefore invalid.
- **Reports:** 114 records contain only `v=DMARC1; p=none`. In total, 124 of the 152 municipalities have not entered a `rua` address and therefore receive no reports.
- **Centralized changes:** The changes occur in clusters. On November 10, 32 Geneva municipalities were added, whose mail servers all run under `acg-geneve.ch`, along with 26 Zurich and 19 Fribourg municipalities. On November 7, there were eight municipalities in Nidwalden and seven in Ticino. The clustering by mail server and canton suggests that IT service providers set the records for all managed municipalities in a single pass.

These records are sufficient for delivery to Outlook.com. They do not yet protect municipal domains against forged senders: Without `rua`, the data needed to raise the policy later is missing. Of the 2,078 municipalities in the index, 537 still have no DMARC record.

## Assessment

Whether Microsoft withdraws the rejection or confirms it as a rule remains open. This matters little for planning: Google and Yahoo have required DMARC from bulk senders since February 2024, Microsoft since May 2025, and Outlook.com's postmaster policies recommend it to all senders. In recent days, a domain without a DMARC record was no longer reliably deliverable to one of the largest mail providers. The record with `p=none` and `rua` can be published in a few minutes and has no side effects on delivery.

## Sources

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Official sender requirements; states the threshold of 5,000 messages per day, error 5.7.515, and the recommendation for all senders.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Support article on the documented error for high-volume senders.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Forum thread containing the moderator response of November 9, 2026, on the ongoing issue; a forum post, not an official service-health notice.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Analysis of a mail provider's outgoing logs starting November 5, 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Hosting provider report with more than 500 rejections on the first day and resolution through `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Operator mailing-list thread with measurements from MXroute, Fastmail, and one.com, including the narrowing down to `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Reproduced test with valid and aligned SPF and DKIM, without a DMARC record.

8.  [heise online: Outlook.com: The email is genuine, yet it still does not arrive](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): German-language summary of the reports from November 9, 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specification, including how to determine the record through the organizational domain.

10.  [Swiss Mail Security Index: Change log](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Daily DNS record changes for 3,364 Swiss organizations; basis for the figures on municipalities.
