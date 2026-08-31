---
title: "Analyze Email Headers Without Uploading the Email: Locally in the Browser Instead of in a Web Tool"
navTitle: "Analyze headers locally"
description: "Email headers contain internal hostnames, IP addresses, and personal data. Anyone who pastes them into an online tool sends this information to a third-party server. Why analysis does not need a server and what a tool running locally in the browser can do."
date: "2026-08-26"
kategorie: "SMTP & Mail Flow"
timeToRead: "7 min read"
themen:
  - smtp-mailflow
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "mail-auth"
  - "troubleshooting"
related:
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "analyze-email-headers-without-uploading-the-email-locally-in-the-browser-instead-of-in-a-web"
translationId: "article-cad792e705cee24e"
translationOf: e-mail-header-analysieren-ohne-upload
url: https://rafaelpfister.ch/en/blog/analyze-email-headers-without-uploading-the-email-locally-in-the-browser-instead-of-in-a-web
translationSourceHash: 11c4e7d120ea34ca557f0136b93120e5e8e9d72dc7350fd2df7880b23ff46649
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:16:04.485Z
translationReview: automatic
---

# Analyze Email Headers Without Uploading the Email: Locally in the Browser Instead of in a Web Tool

The usual way to analyze an email header looks like this: copy the header from the email client, paste it into an online tool, and have it evaluated. That is convenient, but the entire header is sent to the tool operator's server. Few people realize exactly what information they are transmitting.

## What a Header Actually Contains

A complete header from an email in a corporate environment typically contains:

- **Internal hostnames and IP addresses:** Each `Received` line documents a server along the delivery path, including internal Exchange servers, gateways, and load balancers with FQDNs and often private IP addresses. Together, they form a sketch of the email infrastructure.
- **Personal data:** Sender and recipient addresses, display names, the subject line, message IDs, and, depending on the client, the original sender's IP address.
- **Software and versions:** Received lines and product-specific headers identify the products in use, sometimes including version information.
- **Organization-internal assessments:** In Microsoft 365, for example, the complete spam and authentication assessment, tenant identifiers, and the message's internal classification.

For an attacker, this is useful material for preparation; for data protection purposes, it is personal data: the sender, recipient, and subject line of a specific message. Under the revised Data Protection Act, processing by a foreign online tool remains a disclosure to a third party, potentially abroad. With a header from a customer's support case, the issue becomes even more pressing: entering their data into an external web tool is difficult to justify without a legal basis or consent.

## Analysis Does Not Need a Server

The key point is this: a header is plain text, and evaluating it is simply parsing. Sorting the Received chain chronologically, calculating timestamp differences, decoding `Authentication-Results`, comparing domains: none of this requires a server component. Everything runs in JavaScript in the browser, without the header ever leaving the device.

A tool built this way differs fundamentally from a traditional online analyzer in terms of data protection: there is no transmission, no storage by the operator, and no log files containing third-party headers. Analyzing a customer header therefore remains equivalent to opening the file in a local editor, just more readable.

## What a Local Tool Can Do

The [Mail Header Analyzer](/tools/header-analyzer) on this website is built according to this principle. The pasted header is evaluated exclusively locally in the browser. Its functionality shows that nothing is lost in the process:

- **Delivery path with transit times:** The `Received` chain is arranged chronologically, the time spent at each station is calculated, and the longest segment is highlighted. This makes it clear where a slow delivery was actually delayed. Clock skew between servers is detected and reported.
- **Transport encryption per hop:** The TLS version and cipher are read from the Received lines where the receiving server logs them; Microsoft, Postfix, and Exim use different formats.
- **Authentication:** SPF, DKIM, and DMARC results from `Authentication-Results` (RFC 8601), including details such as `header.d`, `smtp.mailfrom`, and Microsoft's `compauth` with reason code.
- **DMARC alignment:** The From domain, envelope-from, and DKIM domain are displayed side by side and assessed according to strict and relaxed alignment.
- **ARC and DKIM integrity:** Dedicated traces in the flow diagram show from where to where the DKIM hash remained intact and from which station onward the ARC chain preserves the verification results.
- **Microsoft environments:** The spam filter fields (`X-Forefront-Antispam-Report`, SCL, CAT) are decoded, and tenant transitions and hybrid classification are marked in the delivery path.

One limitation applies to every header tool, whether local or not: it displays the receiving server's documented assessment, not an independent revalidation. The header cannot tell you whether an SPF record still looks the same today as it did at the time of receipt.

## Context for Other Tools

Some other providers now also evaluate client-side; checking the privacy policy and the browser's network console clarifies whether pasting a header truly results in no request containing the header content. For traditional server-side analyzers, the simple rule applies: do not paste headers from production environments or third parties—at most, use anonymized examples.

For regular analyses of incident or support headers, a locally running tool is therefore the obvious choice: there is no question of where the data ended up.

## Sources

1.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Standard for the Authentication-Results header field, which forms the basis of authentication analysis.

2.  [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321): Definition of Received lines (Trace Information), which can be used to reconstruct the delivery path and transit times.

3.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Reference for Microsoft 365-specific header fields that an analyzer decodes.
