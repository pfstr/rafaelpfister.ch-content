---
title: "Guide for DNS administrators: MX, SPF, DKIM, DMARC and the usual pitfalls"
navTitle: "Email DNS records"
description: "Those who manage a zone usually receive mail records ready-made and only need to publish them. What regularly goes wrong: the 255-byte limit for DKIM, duplicate SPF records, the lookup limit, MX pointing to a CNAME, automatically appended zone suffixes and policies that nobody enforces anymore."
date: "2026-08-04"
kategorie: "SMTP and mail flow"
timeToRead: "15 min read"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
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
slug: "guide-for-dns-administrators-mx-spf-dkim-dmarc-and-the-usual-pitfalls"
translationId: "article-e4699ad7fcea2e20"
translationOf: dns-records-e-mail-stolpersteine
translationSourceHash: e78b63f0179944cf4c36caf7aabfb9ceb0c5dd40d2d3cf1a25e92fee8cbacaca
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:45:09.565Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/guide-for-dns-administrators-mx-spf-dkim-dmarc-and-the-usual-pitfalls
---

# Guide for DNS administrators: MX, SPF, DKIM, DMARC and the usual pitfalls

Those who manage a DNS zone rarely receive mail records written by themselves. The mail team, a provider or a marketing service sends a line with the note that it "only needs to be published". This is exactly where most errors arise, because mail records are the record type where a typo can have two completely different consequences. Either delivery fails immediately and someone gets in touch within minutes, or it continues unchanged and only sender authentication silently fails. The second case regularly goes unnoticed for months until a major recipient places the domain in quarantine.

Since Google and Yahoo tightened their requirements for bulk senders in February 2024 and Microsoft followed in May 2025, there is little tolerance for partially configured domains. SPF, DKIM and a DMARC record are no longer optional for senders above a certain volume, but prerequisites for delivery.

All examples in this article use `example.com` and generic selectors. The values shown are shortened to keep them readable.

## Rules that apply to every mail record

### The 255-byte limit for TXT records

According to RFC 1035, a TXT record consists of one or more `character-strings`, and each individual character string can contain at most 255 bytes. The record as a whole may be longer, but it must then be split into several character strings. Evaluating systems concatenate these parts again without separators.

This becomes practically relevant in exactly one place: DKIM keys with 2048 bits. Their Base64 value is around 400 characters long and does not fit into one character string.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

Most DNS management systems handle this splitting themselves when the value is entered through the regular input field. Anyone who adds quotation marks manually must observe the limit exactly. A wrapped value with a space at the join produces a key that exists syntactically but no longer works cryptographically.

Checking afterwards is important, because an incorrectly assembled key looks entirely inconspicuous in the GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | Outputs only the record values, without headers and metadata |
| `TXT selector1._domainkey.example.com` | Record type and name of the DKIM key record |
| `tr -d '" '` | Removes quotation marks and spaces, thus joining the partial character strings as a verifier reads them |
| `wc -c` | Counts the characters in the combined value; the length must match the template |

</details>

### One record per purpose

SPF and DMARC are defined so that exactly one matching record may exist at a name. With SPF, two `v=spf1` records result in a `permerror`, meaning the check is considered failed, not passed. With DMARC, recipients ignore the domain entirely if multiple records begin with `v=DMARC1`: instead of a strict policy, none applies at all.

This is by far the most common error in established zones. A new service provider is connected, someone adds “their” SPF record instead of extending the existing one, and from that moment the check fails for every sender. Therefore, always check what is already there before adding a new record:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | Outputs only the record values, without headers and metadata |
| `TXT` | Queried record type |
| `example.com`, `_dmarc.example.com` | Queried names: the domain itself for SPF, the `_dmarc` name for DMARC |
| `grep -i spf1` | Filters out the SPF line; `-i` ignores upper and lower case |

</details>

The opposite applies to DKIM: one record is intended per selector, and multiple selectors side by side are normal because each sending service brings its own key.

### The zone suffix in web interfaces

In Infoblox, Windows DNS and virtually all hosting interfaces, the zone name is automatically appended to the entered name. Anyone entering the fully qualified name in the “Name” field gets a record twice as long as intended:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

In the zone file, the equivalent is the missing trailing dot. `mail.example.com` without a dot at the end is a relative name and has the zone name appended; `mail.example.com.` with a dot is absolute. For MX and CNAME targets, this one dot determines whether the domain can be reached.

### Copy-paste is the most common source of errors

Mail record values are almost never typed, but copied from a PDF, ticket, Excel cell or chat. This causes damage that remains invisible in the input field:

- A duplicate `p=` at the start of the DKIM key because the prefix was set twice during assembly. The value `v=DKIM1;k=rsa;p=p=MIIBIjAN...` occurs regularly in practice and produces an unusable key.
- Typographic quotation marks from Word instead of straight ones.
- Non-breaking spaces from PDF layouts that look like ordinary spaces.
- Line breaks in the middle of the Base64 block when the value spanned multiple lines in the PDF.

Base64 knows only the characters A to Z, a to z, 0 to 9, `+`, `/` and `=` as padding characters. Anything else in the `p=` part is an error. A short filter before entering it saves later troubleshooting:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `'%s' "$KEY"` | Outputs the key value unchanged and without an appended line break |
| `tr -d 'A-Za-z0-9+/='` | Removes all characters valid for Base64; only foreign characters remain |
| `wc -c` | Counts the remaining characters |

</details>

If the result is anything other than `0`, the key contains foreign characters.

### Lower the TTL before changes

Before any planned change to an MX, SPF or DKIM record, lower the TTL for a few hours, typically to 300 seconds. Otherwise, depending on the zone, the old value may remain in external resolvers for a day or longer, and a rollback takes just as long. After the change and an observation period, set the TTL back to its regular value.

## MX

The MX record determines which host accepts mail for the domain. There are two rules that are regularly violated.

**The target must be a hostname with an A or AAAA record.** Neither an IP address nor a CNAME is permitted. RFC 2181 explicitly states that the target of an MX record must not be an alias. In practice, it still works with many recipients but not with others, causing error patterns that seemingly affect only individual senders.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**The number is a preference, not a weighting.** The lower value is tried first. A second MX with a high value is useful only if that system knows the same recipient filter. Backup MX entries on systems without recipient validation are a popular spam target because attackers deliberately address the weakest entry.

Domains that only send or have nothing to do with mail receive a null MX according to RFC 7505. It signals that the domain does not accept mail and results in an immediate, unambiguous rejection instead of timeouts:

```text
example.com.  IN  MX  0 .
```

However, a null MX does not replace an SPF or DMARC record. Not receiving mail does not mean nobody sends in your name. Parked subdomains in particular are used for spoofing because they are rarely monitored.

## A, AAAA, PTR and the HELO name

The PTR record for the outgoing IP address is not in your zone, but in the `in-addr.arpa` zone of the provider that owns the address block. It must therefore be requested from the provider rather than set yourself. Many major recipients require the PTR and associated forward record to match, meaning that the name in the PTR must resolve back to the same IP address.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `+short` | Outputs only the record values, without headers and metadata |
| `-x 192.0.2.10` | Reverse query: dig itself creates the PTR name in the `in-addr.arpa` zone |
| `A mail1.example.com` | Forward lookup of the name from the PTR, to verify that the round trip leads to the same IP address |

</details>

The name your mail server states in HELO or EHLO should be the same and should also resolve. A gateway that identifies itself as `localhost.localdomain` or with an internal name is rated less favourably by larger recipients.

Take care when adding an AAAA record. As soon as the mail server is reachable and sends over IPv6, the same requirements apply as for IPv4, in some respects even stricter ones. Google requires a valid PTR for sending IPv6 addresses. If it is missing, sending is rejected even though it worked perfectly over IPv4. An AAAA record on the mail server is therefore never merely a DNS change.

## SPF

SPF specifies which systems may send on behalf of the domain. The record is placed as TXT at the domain itself.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### The lookup limit

Evaluating an SPF record may trigger no more than ten DNS-querying mechanisms. `include`, `a`, `mx`, `ptr`, `exists` and `redirect` are counted recursively: every `include` includes the lookups of the referenced record. `ip4`, `ip6` and `all` are not counted.

If the limit is exceeded, the result is a `permerror`. For DMARC, this means SPF fails, regardless of whether the sending server would actually be authorised. The tricky part is that the error often arises without any action on your part, because an included provider expands its record. Your own record has not changed, yet delivery still suffers.

In addition, at most two “void lookups” are permitted, meaning queries without a result. An `include` pointing to a domain that no longer exists counts towards this. References to retired providers should therefore be removed, not retained as a precaution.

### What does not belong in an SPF record

- **`ptr`** is specified but considered obsolete since RFC 7208 and should not be used. Evaluating systems may ignore it.
- **`+all`** authorises any sender and is therefore more harmful than having no SPF record at all.
- **`?all`** is neutral and thus practically worthless for DMARC.
- **A separate SPF record (type 99)** is no longer needed. It was retired by RFC 7208; SPF is exclusively in TXT.

The choice between `~all` (softfail) and `-all` (hardfail) depends on how completely sending paths are covered. As long as there is doubt, `~all` is the right choice. Those already enforcing DMARC and evaluating reports can move to `-all`.

### Subdomains inherit nothing

An SPF record at `example.com` does not apply to `newsletter.example.com`. Every sending subdomain needs its own record. For all others, a wildcard entry is recommended to make clear that nothing originates there:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Caution: a TXT wildcard also answers queries for names such as `_dmarc.sub.example.com` if no explicit record exists there. This is usually harmless, but it can confuse troubleshooting because every TXT query receives an answer.

### SPF flattening

Tools that resolve all `include` references and replace them with the underlying IP addresses solve the lookup limit at the cost of maintainability. If the provider changes its addresses, sending fails, and nobody notices because everything in the local record seemingly looks correct. Anyone taking this route therefore needs an automated comparison that regularly checks the list against the source. As a one-time manual task, this method will fail sooner or later.

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

The provider can then rotate its key independently without anyone needing to act in your zone. Otherwise, this rotation is regularly neglected because it requires coordination between two teams. However, a CNAME excludes every additional record at the same name; this is a fundamental DNS rule, not a DKIM peculiarity.

### Rotation without downtime

When changing keys, first publish the new selector, then switch the sending server to it, and only then remove the old record. Anyone deleting the old key immediately invalidates signatures on all messages still in transit or queues and makes retrospective verification impossible. A lead time of several days between switching and deletion is appropriate.

A record with an empty `p=` is not a broken entry, by the way, but the specified way to mark a key as revoked.

### Key length

1024 bits are considered obsolete; 2048 bits are the standard. Larger RSA keys offer virtually no additional benefit and only increase the likelihood that an intermediate system will not process the record correctly.

## DMARC

DMARC combines SPF and DKIM with an instruction on what should happen when a check fails, and returns reports. The record is located under `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Since May 2026, RFC 9989 and the reporting specifications RFC 9990 and RFC 9991 have introduced the revised version, replacing RFC 7489. Three changes are important in practice:

- **`pct` has been removed.** Phased rollout through a percentage no longer exists. It is replaced by `t=y`, which marks the domain as being in testing: reports continue, but the policy should not be enforced.
- **`np` is new.** It sets the policy for non-existent subdomains, closing a gap attackers like to exploit because invented subdomains were previously covered only by `sp`. Without an explicit value, `np` follows the value of `sp`.
- **The Public Suffix List has been replaced by a `Tree Walk`.** The organisational domain is no longer determined from an externally maintained list, but through staged DNS queries along the name tree. This noticeably changes evaluation for large namespaces with many levels.

### Alignment is the real core

DMARC does not pass because SPF or DKIM technically passed, but only if at least one of them also matches the visible sender domain in the `From` header. SPF is checked against the envelope sender domain, which regularly differs for forwarding, newsletter services and ticket systems. This is exactly why messages with valid SPF occasionally fail the DMARC check.

With `adkim=r` and `aspf=r` (relaxed, the default), a match at organisational-domain level is sufficient. `s` requires exact equality including the subdomain and in practice almost always fails on one of the sending paths.

### External report addresses require authorisation

If reports are to go to an address outside your own domain, for example to a DMARC analytics service, the receiving domain must authorise this. Without this record, many recipients simply send nothing, leaving the analysis empty while everything in your own record looks correct:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

This entry is created by the operator of the destination zone, not by you. Commercial services do this automatically, but a self-operated collection mailbox in another domain you own does not.

### Typical syntax errors

Tag names and policy values must be lowercase; `p=Reject` is invalid. Tags are separated by a semicolon; a missing separator makes the rest of the line ineffective. And `p` must be the first tag after `v`. A record consisting only of `v=DMARC1; rua=...` contains no policy and is incomplete.

### The rollout

`p=none` is a measurement state, not a goal. It changes nothing in how recipients handle your mail and serves only to identify all legitimate sending paths through reports. Anyone who does not move from `quarantine` to `reject` within a few months after introduction has made the effort without gaining the protection. The organisational side of this process, including a decision template, is a separate topic and described in the DMARC blueprint.

## MTA-STS and TLS-RPT

SMTP encrypts opportunistically: if the other side offers STARTTLS, encryption is used; otherwise it is not. An attacker in a position to manipulate traffic can remove the STARTTLS announcement and keep the connection in plaintext. MTA-STS closes this gap for receiving domains.

MTA-STS consists of two parts, only one of which is in DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

The actual policy is a file located at `https://mta-sts.example.com/.well-known/mta-sts.txt` and must be delivered via a valid certificate:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Almost all sources of error lie outside the zone:

- **The `id` must change with every policy change.** It is the only indication for sending systems that a new policy must be fetched. Anyone adapting the file while leaving the `id` unchanged works against cached copies until `max_age` expires.
- **The MX list in the policy and the MX records must match.** A new MX missing from the policy is rejected by senders with `mode: enforce`. During migrations, the policy must therefore be adapted before the MX change.
- **`mode: testing` first.** In this mode, violations are only reported, not enforced. Switch to `enforce` when the reports are clean.
- **A CAA record can block certificate issuance for the policy host** if it specifies a different certification authority than the one being used.

TLS-RPT provides the associated reports and is a single record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT is useful even without MTA-STS because it makes failed transport encryption visible in the first place.

## DANE

DANE achieves the same goal as MTA-STS but anchors trust in DNS rather than the web PKI. It requires a zone signed end to end with DNSSEC; without DNSSEC, a TLSA record is ineffective.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Operationally, the key point is that the TLSA record must be correct before every certificate change. The usual procedure publishes the new hash alongside the old one, then changes the certificate and finally removes the old entry. Reversing this order makes the mail server unreachable for all DANE-validating senders, including major German-speaking providers. In Switzerland, DANE is encountered much less frequently than MTA-STS, usually due to the zone lacking DNSSEC signing.

## BIMI

BIMI displays the brand logo in the inbox and is the only mechanism covered here that is not yet an RFC but continues to be maintained as an Internet Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

The requirements are high: an enforced DMARC policy with `quarantine` or `reject`, a logo in SVG Tiny Portable/Secure format, and for most providers, a paid Verified Mark Certificate. BIMI is therefore not a security mechanism but a visibility issue, and it belongs at the end of the sequence, not at the beginning.

## Other related records

**Autodiscover and SRV:** Exchange environments use `autodiscover.example.com` as a CNAME or an SRV record `_autodiscover._tcp.example.com`. Both concern client configuration rather than mail flow, but are often overlooked during migration and then result in profiles that can no longer be configured.

**CAA:** It has nothing directly to do with mail, but determines which certification authority may issue a certificate for `mta-sts.example.com` or the mail server name.

**Split-horizon zones:** Where an internal DNS zone has the same name as the public one, mail records often do not exist internally. Internal systems performing SPF or DKIM checks then reach different results than the outside world. Every mail-record change should therefore include the question of whether the internal zone must be updated.

## A few quick tests

Deliberately send all queries to a public resolver so that neither the internal cache nor a split-horizon zone responds:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `@1.1.1.1` | Sends the query to this resolver rather than the one configured in `/etc/resolv.conf` |
| `+short` | Outputs only the record values, without headers and metadata |
| `MX`, `TXT` | Queried record types |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Names defined below the domain for DMARC, DKIM, MTA-STS and TLS-RPT |

</details>

Against the authoritative server to bypass caching completely:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `NS example.com` | Determines the authoritative nameservers of the zone |
| `@ns1.example.com` | Sends the subsequent query directly to one of these authoritative servers |
| `+norecurse` | Does not set the Recursion-Desired bit; the server responds only from its own zone data, not from a cache |

</details>

On Windows without `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-type=TXT` | Record type to query |
| `_dmarc.example.com` | Queried name |
| `1.1.1.1` | Resolver to use instead of the system-wide configured one |

</details>

For complete evaluation including SPF lookup counting, DKIM selector lookup and alignment checking, this page provides the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check), which checks a domain against all records described here in one pass.

However, the most meaningful test remains a real message. Send an email to a mailbox at a major provider and inspect the `Authentication-Results` line in the header. It shows in one line what SPF, DKIM and DMARC actually returned, replacing any theory about the zone file.

## Order during a migration

When changing mail providers, the following sequence has proven effective:

1. Lower the TTL of all affected records to 300 seconds, at least one day in advance.
2. Publish the new provider’s DKIM selectors while the old ones are still present.
3. Extend SPF with the new provider without removing the old one, and recalculate the lookup limit.
4. For MTA-STS, adapt the policy to the new MX names and increase the `id` before changing the MX records.
5. Switch the MX records and monitor delivery.
6. Only after several days without complaints, remove the old SPF includes and DKIM selectors.
7. Reset the TTL.

The most common problem in this process is performing step 6 too early: old entries are deleted together with the switch, and everything still using the previous path fails sender authentication.

## Conclusion

Mail records differ from all other DNS entries in that an error does not necessarily become apparent. An incorrect A record leads to a ticket within minutes, whereas a duplicate SPF record or a DKIM key with one character too many can result in a delivery rate that slowly declines over weeks.

Three rules prevent most of these cases. First: before every new record, check what is already present rather than placing a second record beside it. Second: after every change, verify against a public resolver and compare the value character by character with the template, not merely visually. Third: during changes, always publish the new entry first, then switch, then remove the old one. Those who follow this sequence always retain a way back with mail records.

## Change history

- **25 October 2026:** Updated the source link to Microsoft’s new address.

## Kilder

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Defines, among other things, the 255-byte limit of an individual `character-string` in TXT records.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): States in section 10.3 that the target of an MX record must not be an alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Lookup limit of ten mechanisms, void lookup limit, retirement of the SPF RR type and recommendation against the `ptr` mechanism.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Structure of the key record under `_domainkey`, the selector’s meaning and the empty `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Current DMARC specification from May 2026, replacing RFC 7489; removal of `pct`, new `np` tag, Tree Walk instead of Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format and delivery of aggregate reports, including authorisation of external recipient domains.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identification of domains that do not accept mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS record, policy file, meaning of `id` and the modes `testing` and `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Structure of the `_smtp._tls` record and reports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA records for SMTP and the requirement for a DNSSEC-signed zone.

11.  [Brand Indicators for Message Identification (BIMI), Internet Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Current status of the BIMI specification, still not an RFC.

12.  [Google: Email sender guidelines](https://support.google.com/a/answer/81126): Requirements for senders, including the PTR requirement for sending IPv6 addresses and requirements for bulk senders in force since February 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399730): Requirements for senders of 5,000 messages per day or more, effective since May 2025.
