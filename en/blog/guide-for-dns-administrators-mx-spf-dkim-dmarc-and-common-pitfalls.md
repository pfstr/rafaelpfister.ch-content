---
title: "Guide for DNS Administrators: MX, SPF, DKIM, DMARC, and Common Pitfalls"
navTitle: "Email DNS Records"
description: "Anyone managing a zone is usually given mail records ready-made and only needs to publish them. What regularly goes wrong: the 255-byte limit for DKIM, duplicate SPF records, the lookup limit, MX pointing to a CNAME, the automatically appended zone suffix, and policies that nobody enforces anymore."
date: "2026-08-04"
kategorie: "SMTP and Mail Flow"
timeToRead: "15 min read"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
produkte:
  - "uebergreifend"
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - smtp-verbindung-testen-linux
  - ghost-sender-exchange-online-nebeneingang
slug: "guide-for-dns-administrators-mx-spf-dkim-dmarc-and-common-pitfalls"
translationId: "article-e4699ad7fcea2e20"
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
url: https://rafaelpfister.ch/en/blog/guide-for-dns-administrators-mx-spf-dkim-dmarc-and-common-pitfalls
translationSourceHash: dc806bed491a47ecc1118249566d9303b0201f4bdb5153a966385a7c9373b31f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-07T05:09:55.781Z
translationReview: automatic
---

# Guide for DNS Administrators: MX, SPF, DKIM, DMARC, and Common Pitfalls

Anyone managing a DNS zone is rarely handed mail records they wrote themselves. The mail team, a provider, or a marketing service sends a line with the note that it "just needs to be published." This is exactly where most errors originate, because mail records are the type of record where a typo can have two entirely different consequences. Either delivery fails immediately and someone reports it within minutes, or everything continues unchanged and only sender authentication silently fails. The second case regularly goes unnoticed for months until a major recipient places the domain in quarantine.

Since Google and Yahoo tightened their requirements for bulk senders in February 2024, and Microsoft followed suit in May 2025, tolerance for partially configured domains has become very low. For senders above a certain volume, SPF, DKIM, and a DMARC record are no longer optional—they are delivery requirements.

All examples in this article use `example.com` and generic selectors. The values shown are shortened to keep them readable.

## Rules That Apply to Every Mail Record

### The 255-Byte Limit for TXT Records

According to RFC 1035, a TXT record consists of one or more `character-strings`, and each individual character string can hold a maximum of 255 bytes. The record as a whole may be longer, but it must then be split into multiple character strings. Evaluating systems concatenate these parts again without separators.

This becomes practically relevant in exactly one place: DKIM keys with 2048 bits. Their Base64 value is around 400 characters long and does not fit in one character string.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

Most DNS management systems handle this split automatically when the value is entered through the regular input field. If you add quotation marks manually instead, you must observe the limit exactly. A split value with a space at the join creates a key that exists syntactically but no longer matches cryptographically.

Checking afterward is important, because an incorrectly assembled key looks completely unremarkable in the GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### One Record per Purpose

SPF and DMARC are defined so that exactly one matching record may exist at a name. For SPF, two `v=spf1` records result in a `permerror`, meaning the check fails rather than passes. For DMARC, recipients ignore the domain entirely if multiple records begin with `v=DMARC1`: instead of a strict policy, no policy applies at all.

This is by far the most common error in mature zones. A new provider is connected, someone adds "their" SPF record rather than extending the existing one, and from that moment on authentication fails for every sender. Before adding any new record, always check what is already there:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

The opposite applies to DKIM: one record per selector is intended, and multiple selectors side by side are normal because every sending service brings its own key.

### The Zone Suffix in Web Interfaces

In Infoblox, Windows DNS, and virtually all hosting interfaces, the zone name is automatically appended to the entered name. If you enter the fully qualified name in the "Name" field, you get a record twice as long as intended:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

In a zone file, the counterpart is the missing trailing period. `mail.example.com` without a period at the end is a relative name and has the zone name appended; `mail.example.com.` with a period is absolute. For MX and CNAME targets, this single period determines whether the domain is reachable.

### Copying and Pasting Is the Most Common Source of Errors

Mail record values are almost never typed; they are copied from a PDF, ticket, Excel cell, or chat. This causes damage that remains invisible in the input field:

- A duplicate `p=` at the start of the DKIM key because the prefix was added twice during assembly. The value `v=DKIM1;k=rsa;p=p=MIIBIjAN...` is a real classic and results in an unusable key.
- Typographic quotation marks from Word instead of straight ones.
- Nonbreaking spaces from PDF layouts that look like normal spaces.
- Line breaks in the middle of the Base64 block when the value spanned multiple lines in the PDF.

Base64 recognizes only the characters A through Z, a through z, 0 through 9, `+`, `/`, and `=` as padding characters. Anything else in the `p=` part is an error. A short filter before entering it saves troubleshooting later:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

If this returns anything other than `0`, the key contains foreign characters.

### Lower the TTL Before Changes

Before any planned change to an MX, SPF, or DKIM record, lower the TTL for a few hours to a low value, typically 300 seconds. Otherwise, depending on the zone, the old value may remain in external resolvers for a day or longer, and a rollback takes just as long. After the change and an observation period, reset the TTL to its regular value.

## MX

The MX record specifies which host accepts mail for the domain. There are two rules that are regularly violated.

**The target must be a hostname with an A or AAAA record.** Neither an IP address nor a CNAME is permitted. RFC 2181 explicitly states that the target of an MX record must not be an alias. In practice, this still works with many recipients but not with others, leading to failure patterns that seemingly affect only individual senders.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**The number is a preference, not a weight.** The lower value is tried first. A second MX with a high number is only useful if that system applies the same recipient filtering. Backup MX entries pointing to systems without recipient validation are popular spam targets because attackers deliberately target the weakest entry.

Domains that only send or have nothing to do with mail receive a Null MX according to RFC 7505. It signals that the domain does not accept mail and produces an immediate, unambiguous rejection instead of timeouts:

```text
example.com.  IN  MX  0 .
```

However, a Null MX does not replace an SPF or DMARC record. Not receiving mail does not mean nobody sends in your name. Parked subdomains in particular are used for spoofing because nobody rarely looks there.

## A, AAAA, PTR, and the HELO Name

The PTR record for the outgoing IP address is not located in your zone but in the provider's `in-addr.arpa` zone, where the provider owns the address block. It must therefore be requested from the provider rather than set yourself. Many major recipients require the PTR and its associated forward record to match, meaning the name from the PTR must resolve back to the same IP address.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

The name your mail server gives in HELO or EHLO should be the same and should also resolve. A gateway that identifies itself as `localhost.localdomain` or with an internal name receives a worse rating from major recipients.

Use caution when adding an AAAA record. As soon as the mail server becomes reachable and sends over IPv6, the same requirements as for IPv4 apply, and in some respects even stricter ones. Google requires a valid PTR for sending IPv6 addresses. If it is missing, sending is rejected even though it worked perfectly over IPv4. An AAAA record on a mail server is therefore never just a DNS change.

## SPF

SPF defines which systems are permitted to send on behalf of the domain. The record is placed as TXT at the domain itself.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### The Lookup Limit

Evaluating an SPF record may trigger no more than ten DNS-querying mechanisms. `include`, `a`, `mx`, `ptr`, `exists` and `redirect` are counted recursively: every `include` brings along the lookups of the included record. `ip4`, `ip6`, and `all` are not counted.

If the limit is exceeded, the result is a `permerror`. For DMARC, this means SPF does not pass, regardless of whether the sending server would actually be authorized. The tricky part is that this error often occurs without any action on your part, because an included provider expands its record. Your own record has not changed, yet delivery still drops off.

Additionally, no more than two "void lookups" are permitted—queries without a result. An `include` pointing to a domain that no longer exists counts toward this limit. References to retired providers should therefore be removed, not kept just in case.

### What Does Not Belong in an SPF Record

- **`ptr`** is specified but has been obsolete since RFC 7208 and should not be used. Evaluating systems may ignore it.
- **`+all`** authorizes any sender and is therefore more harmful than having no SPF record at all.
- **`?all`** is neutral and therefore practically worthless for DMARC.
- **A separate SPF record (type 99)** is no longer needed. It was abolished in RFC 7208; SPF belongs exclusively in TXT.

The choice between `~all` (softfail) and `-all` (hardfail) depends on how fully the sending paths have been captured. As long as there is any doubt, `~all` is the right choice. If you already enforce DMARC and evaluate the reports, you can move to `-all`.

### Subdomains Do Not Inherit Anything

An SPF record at `example.com` does not apply to `newsletter.example.com`. Every sending subdomain needs its own record. For all others, a wildcard entry is recommended to make it clear that nothing is sent from them:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Caution: a TXT wildcard also answers queries for names such as `_dmarc.sub.example.com` if no explicit record exists there. This is usually harmless, but it can confuse troubleshooting because every TXT query receives an answer.

### SPF Flattening

Tools that resolve all `include` references and replace them with the IP addresses behind them solve the lookup limit at the cost of maintainability. If the provider changes its addresses, sending fails, and nobody notices because everything appears correct in the local record. Anyone taking this approach therefore needs an automated comparison that regularly checks the list against the source. As a one-time manual task, this method will eventually fail.

## DKIM

DKIM signs outgoing messages. The public key is located under `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

The selector can be chosen freely and is specified by the sending system. A descriptive name with a date makes later rotation much easier than `s1` and `s2`.

### Delegation via CNAME

Where the sending service offers it, the CNAME variant is preferable to a direct entry:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

The provider can then rotate its key independently without anyone needing to make changes in your zone. Otherwise, this rotation regularly gets neglected because it requires coordination between two teams. However, a CNAME excludes every additional record at the same name; this is a basic DNS rule, not a DKIM peculiarity.

### Rotation Without Downtime

When changing keys, first publish the new selector, then switch the sending server to it, and only afterward remove the old record. Deleting the old key immediately invalidates the signatures of all messages still in transit or in queues and makes later verification impossible. A lead time of a few days between the switch and deletion is appropriate.

A record with an empty `p=` is not a broken entry, by the way, but the specified way to mark a key as revoked.

### Key Length

1024 bits are considered obsolete; 2048 bits are the standard. Larger RSA keys provide virtually no additional benefit and only increase the likelihood that an intermediary system will not process the record cleanly.

## DMARC

DMARC combines SPF and DKIM with an instruction on what should happen when authentication fails, and it returns reports. The record is located under `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Since May 2026, the revised version has applied with RFC 9989 and the reporting specifications RFC 9990 and RFC 9991, replacing RFC 7489. Three changes are important in practice:

- **`pct` has been removed.** Phased introduction through a percentage no longer exists. It is replaced by `t=y`, which marks the domain as being in testing: reports continue, but the policy should not be enforced.
- **`np` is new.** It sets the policy for nonexistent subdomains, closing a gap attackers like to exploit because invented subdomains were previously covered only through `sp`. Without an explicit value, `np` follows the value of `sp`.
- **The Public Suffix List has been replaced by a `Tree Walk`.** The organizational domain is no longer determined from an externally maintained list, but through stepped DNS queries along the name tree. This noticeably changes evaluation for large namespaces with many levels.

### Alignment Is the Real Core

DMARC does not pass merely because SPF or DKIM technically passed. At least one of them must additionally match the visible sender domain in the `From` header. SPF is checked against the envelope sender domain, which regularly differs for forwarding, newsletter services, and ticketing systems. This is exactly why messages with valid SPF occasionally fail the DMARC check.

With `adkim=r` and `aspf=r` (relaxed, the default), matching at the organizational-domain level is sufficient. `s` requires an exact match including the subdomain and almost always fails in practice for one of the sending paths.

### External Report Addresses Need Authorization

If reports are to be sent to an address outside your own domain, such as a DMARC analysis service, the receiving domain must authorize it. Without this record, many recipients simply send nothing, leaving the analysis empty even though everything looks correct in your own record:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

This entry is created by the operator of the destination zone, not by you. Commercial services do this automatically, but a self-operated collection mailbox in another one of your own domains does not.

### Common Syntax Errors

Tag names and policy values must be lowercase; `p=Reject` is invalid. Tags are separated by semicolons; a missing separator renders the rest of the line ineffective. And `p` must be the first tag after `v`. A record consisting only of `v=DMARC1; rua=...` contains no policy and is incomplete.

### The Rollout

`p=none` is a measurement state, not a goal. It does not change how recipients handle your mail and serves only to identify all legitimate sending paths through the reports. Anyone who does not move from `quarantine` to `reject` within a few months after introduction has made the effort without gaining the protection. The organizational side of this process, including a decision template, is a topic of its own and is described in the DMARC Blueprint.

## MTA-STS and TLS-RPT

SMTP encrypts opportunistically: if the remote side offers STARTTLS, encryption is used; otherwise it is not. An attacker in a position to manipulate traffic can remove the STARTTLS announcement and thereby keep the connection in clear text. MTA-STS closes this gap for receiving domains.

MTA-STS consists of two parts, and only one of them is in DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

The actual policy is a file located under `https://mta-sts.example.com/.well-known/mta-sts.txt` and must be served through a valid certificate:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Almost all pitfalls are outside the zone:

- **The `id` must change with every policy change.** It is the only indication to sending systems that a new policy must be retrieved. If you modify the file and leave the `id` unchanged, you are working against cached copies until `max_age` expires.
- **The MX list in the policy and the MX records must match.** A new MX missing from the policy is rejected by senders using `mode: enforce`. During migrations, the policy must therefore be updated before the MX change.
- **`mode: testing` first.** In this mode, violations are only reported, not enforced. Switch to `enforce` when the reports are clean.
- **A CAA record can block certificate issuance for the policy host** if it specifies a different certificate authority than the one being used.

TLS-RPT provides the corresponding reports and is a single record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT is useful even without MTA-STS because it makes failed transport encryption visible in the first place.

## DANE

DANE achieves the same goal as MTA-STS, but anchors trust in DNS rather than the web PKI. It requires a zone signed end-to-end with DNSSEC, and without DNSSEC, a TLSA record is ineffective.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Operationally, the key point is this: before every certificate change, the TLSA record must already match. The standard procedure publishes the new hash alongside the old one, then changes the certificate, and then removes the old entry. Reversing this order makes the mail server unreachable for all DANE-validating senders, including major German-speaking providers. In Switzerland, DANE is encountered significantly less often than MTA-STS, usually because the zone lacks DNSSEC signing.

## BIMI

BIMI displays the brand logo in the inbox and is the only mechanism covered here that is not yet an RFC but continues to be maintained as an Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

The requirements are high: an enforced DMARC policy with `quarantine` or `reject`, a logo in SVG Tiny Portable/Secure format, and, for most providers, a paid Verified Mark Certificate. BIMI is therefore not a security mechanism but a visibility matter, and it belongs at the end of the sequence, not the beginning.

## Other Related Records

**Autodiscover and SRV:** Exchange environments use `autodiscover.example.com` as a CNAME or an SRV record `_autodiscover._tcp.example.com`. Both concern client configuration rather than mail flow, but are often overlooked during migrations and then result in profiles that can no longer be configured.

**CAA:** It has nothing directly to do with mail, but it determines which certificate authority may issue a certificate for `mta-sts.example.com` or the mail server name.

**Split-horizon zones:** Where an internal DNS zone has the same name as the public one, mail records often do not exist internally. Internal systems that perform SPF or DKIM checks then reach different results than the outside world. Every change to mail records should therefore include the question of whether the internal zone needs to be updated.

## Some Quick Tests

Run all queries deliberately against a public resolver so that neither the internal cache nor a split-horizon zone responds:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Against the authoritative server to bypass caching completely:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

On Windows without `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

For a complete evaluation including SPF lookup counting, DKIM selector lookup, and alignment verification, this page offers the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check), which checks a domain in one pass against all records described here.

However, the most meaningful test remains a real message. Send an email to a mailbox with a major provider and inspect the `Authentication-Results` line in the header. In one line, it shows what SPF, DKIM, and DMARC actually produced, replacing any theory about the zone file.

## Order of Operations During a Migration

When changing mail providers, this sequence has proven effective:

1. Lower the TTL of all affected records to 300 seconds, at least one day in advance.
2. Publish the new provider's DKIM selectors while the old ones are still in place.
3. Extend SPF with the new provider without removing the old one, and recalculate the lookup limit.
4. For MTA-STS, update the policy with the new MX names and increase the `id` before changing the MX records.
5. Switch the MX records and monitor delivery.
6. Only after several days without issues, remove the old SPF includes and DKIM selectors.
7. Reset the TTL.

The most common issue in this process is performing step 6 too early: old entries are deleted together with the switch, and everything still using the previous path fails sender authentication.

## Conclusion

Mail records differ from all other DNS entries because an error does not necessarily become apparent. An incorrect A record leads to a ticket within minutes, while a duplicate SPF record or a DKIM key with one character too many instead causes a delivery rate that slowly declines over weeks.

Three rules prevent most of these cases. First: before adding any record, check what already exists rather than placing a second one beside it. Second: after every change, verify against a public resolver and compare the value character by character against the template, not just visually. Third: during changes, always publish the new configuration first, then switch over, then remove the old configuration. Anyone following this order always has a way back with mail records.

## Sources

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Defines, among other things, the 255-byte limit of an individual `character-string` in TXT records.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): States in section 10.3 that the target of an MX record must not be an alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Lookup limit of ten mechanisms, void lookup limit, abolition of the SPF RR type, and recommendation against the `ptr` mechanism.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Structure of the key record under `_domainkey`, meaning of the selector and the empty `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Current DMARC specification from May 2026, replacing RFC 7489; removal of `pct`, new `np` tag, Tree Walk instead of the Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format and delivery of aggregate reports, including authorization of external recipient domains.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identification of domains that do not accept mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS record, policy file, meaning of `id` and the `testing` and `enforce` modes.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Structure of the `_smtp._tls` record and the reports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA records for SMTP and the requirement for a DNSSEC-signed zone.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Current state of the BIMI specification, still not an RFC.

12.  [Google: Email Sender Guidelines](https://support.google.com/a/answer/81126): Sender requirements, including the PTR requirement for sending IPv6 addresses and the requirements for bulk senders in effect since February 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requirements for senders of 5000 messages per day or more, effective since May 2025.
