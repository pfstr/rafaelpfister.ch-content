---
title: "Guide for DNS administrators: MX, SPF, DKIM, DMARC and the usual pitfalls"
navTitle: "Email DNS records"
description: "Anyone managing a zone will usually receive mail records ready-made and only need to publish them. What regularly goes wrong: the 255-byte limit for DKIM, duplicate SPF records, the lookup limit, an MX pointing to a CNAME, the automatically appended zone suffix and policies that nobody enforces anymore."
date: "2026-08-04"
kategorie: "SMTP and mail flow"
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
slug: "guide-for-dns-administrators-mx-spf-dkim-dmarc-and-the-usual-pitfalls"
translationId: "article-e4699ad7fcea2e20"
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
url: https://rafaelpfister.ch/no/blog/guide-for-dns-administrators-mx-spf-dkim-dmarc-and-the-usual-pitfalls
translationSourceHash: dc806bed491a47ecc1118249566d9303b0201f4bdb5153a966385a7c9373b31f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:07:04.562Z
translationReview: required
---

# Guide for DNS administrators: MX, SPF, DKIM, DMARC and the usual pitfalls

Anyone managing a DNS zone rarely receives mail records that they wrote themselves. The mail team, a provider or a marketing service sends a line with the note that it "just needs to be published". This is exactly where most errors arise, because mail records are the type of record where a typo can have two completely different consequences. Either delivery fails immediately and someone gets in touch within minutes, or it continues unchanged while only sender authentication silently fails. The second case regularly goes unnoticed for months, until a major recipient puts the domain into quarantine.

Since Google and Yahoo tightened their requirements for bulk senders in February 2024 and Microsoft followed suit in May 2025, tolerance for half-configured domains has become low. SPF, DKIM and a DMARC record are no longer optional for senders above a certain volume, but prerequisites for delivery.

All examples in this article use `example.com` and generic selectors. The values shown are abbreviated to keep them readable.

## Rules that apply to every mail record

### The 255-byte limit for TXT records

According to RFC 1035, a TXT record consists of one or more `character-strings`, and any single such character string can contain at most 255 bytes. The record as a whole may be longer, but it must then be split into multiple character strings. Evaluating systems concatenate these parts again without separators.

This becomes relevant in practice in exactly one place: DKIM keys with 2048 bits. Their Base64 value is around 400 characters long and does not fit into one character string.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

Most DNS management systems handle this splitting themselves when the value is entered through the regular input field. Anyone manually adding quotation marks must adhere to the limit exactly. A wrapped value with a space at the join produces a key that exists syntactically but no longer matches cryptographically.

Checking afterwards is important, because an incorrectly assembled key looks entirely inconspicuous in the GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### One record per purpose

SPF and DMARC are defined so that exactly one matching record may exist at a name. With SPF, two `v=spf1` records result in a `permerror`, meaning the check is considered to have failed, not passed. With DMARC, recipients ignore the domain entirely if several records begin with `v=DMARC1`: instead of a strict policy, no policy applies at all.

This is by far the most common error in long-established zones. A new service provider is connected, someone adds “their” SPF record instead of extending the existing one, and from that point the check fails for every sender. Before creating a new record, it is therefore essential to check what is already there:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

DKIM is the opposite: one record is intended per selector, and multiple selectors side by side are normal because every sending service has its own key.

### The zone suffix in web interfaces

In Infoblox, Windows DNS and virtually all hosting interfaces, the zone name is automatically appended to the entered name. Anyone entering the fully qualified name in the “Name” field gets a record twice as long as intended:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

In the zone file, the counterpart is the missing trailing dot. `mail.example.com` without a dot at the end is a relative name and has the zone name appended; `mail.example.com.` with a dot is absolute. For MX and CNAME targets, this single dot determines whether the domain is reachable.

### Copy and paste is the most common source of errors

Mail record values are almost never typed, but copied from a PDF, a ticket, an Excel cell or a chat. This can cause damage that remains invisible in the input field:

- A duplicate `p=` at the beginning of the DKIM key because the prefix was added twice during assembly. The value `v=DKIM1;k=rsa;p=p=MIIBIjAN...` is a real classic and results in an unusable key.
- Typographic quotation marks from Word instead of straight ones.
- Non-breaking spaces from PDF layouts that look like ordinary spaces.
- Line breaks in the middle of the Base64 block when the value spanned several lines in the PDF.

Base64 recognises only the characters A to Z, a to z, 0 to 9, `+`, `/` and `=` as padding characters. Anything else in the `p=` part is an error. A short filter before entering it saves later troubleshooting:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

If this returns anything other than `0`, the key contains foreign characters.

### Lower the TTL before changes

Before any planned change to an MX, SPF or DKIM record, the TTL should be set to a low value for a few hours, typically 300 seconds. Otherwise, depending on the zone, the old value remains in external resolvers for a day or longer, and a rollback takes just as long. After the change and an observation period, the TTL is set back to its regular value.

## MX

The MX record specifies which host accepts mail for the domain. There are two rules that are regularly violated.

**The target must be a hostname with an A or AAAA record.** Neither an IP address nor a CNAME is permitted. RFC 2181 explicitly states that the target of an MX record must not be an alias. In practice, it still works with many recipients but not with others, leading to problems that appear to affect only individual senders.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**The number is a preference, not a weighting.** The lower value is tried first. A second MX with a high number only makes sense if that system uses the same recipient filtering. Backup MX entries on systems without recipient validation are a popular target for spam because attackers deliberately target the weakest entry.

Domains that only send mail or have nothing to do with mail at all receive a Null MX under RFC 7505. It signals that the domain accepts no mail and ensures immediate, unambiguous rejection rather than timeouts:

```text
example.com.  IN  MX  0 .
```

However, a Null MX does not replace an SPF or DMARC record. Not receiving mail does not mean nobody is sending in your name. Parked subdomains in particular are used for spoofing because nobody is likely to watch them.

## A, AAAA, PTR and the HELO name

The PTR record for the outgoing IP address is not in your zone, but in the provider’s `in-addr.arpa` zone, where the address block belongs. It must therefore be requested from the provider rather than set yourself. Many major recipients require the PTR and corresponding forward record to match, meaning that the name from the PTR resolves back to the same IP address.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

The name your mail server uses in HELO or EHLO should be the same and must also resolve. A gateway identifying itself as `localhost.localdomain` or using an internal name receives a poorer rating from larger recipients.

Take care when adding an AAAA record. As soon as the mail server becomes reachable and sends over IPv6, the same requirements as for IPv4 apply, in some respects even stricter ones. Google requires a valid PTR for sending IPv6 addresses. If it is missing, sending is rejected even though IPv4 worked flawlessly. An AAAA record on a mail server is therefore never just a DNS change.

## SPF

SPF specifies which systems are permitted to send on behalf of the domain. The record is placed as a TXT record at the domain itself.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### The lookup limit

Evaluating an SPF record may trigger no more than ten DNS-querying mechanisms. Counted are `include`, `a`, `mx`, `ptr`, `exists` and `redirect`, recursively: each `include` brings along the lookups of the included record. `ip4`, `ip6` and `all` are not counted.

If the limit is exceeded, the result is a `permerror`. For DMARC, this means SPF has failed regardless of whether the sending server would actually be authorised. The tricky part is that this often happens through no action of your own because an included provider expands its record. Your own record has not changed, but delivery still drops off.

In addition, no more than two “void lookups” are allowed, meaning queries with no result. An `include` for a domain that no longer exists counts towards this. References to retired providers should therefore be removed, not retained as a precaution.

### What does not belong in an SPF record

- **`ptr`** is specified but considered obsolete since RFC 7208 and should not be used. Evaluating systems may ignore it.
- **`+all`** authorises any sender at all and is therefore more harmful than having no SPF record.
- **`?all`** is neutral and therefore practically worthless for DMARC.
- **A separate SPF record (type 99)** is no longer needed. It was deprecated by RFC 7208; SPF is exclusively in TXT.

The choice between `~all` (softfail) and `-all` (hardfail) depends on how completely the sending paths are covered. As long as there is doubt, `~all` is the right choice. Anyone already enforcing DMARC and evaluating reports can move to `-all`.

### Subdomains inherit nothing

An SPF record at `example.com` does not apply to `newsletter.example.com`. Every sending subdomain needs its own record. For all others, a wildcard entry is recommended to make it clear that nothing originates there:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Caution: a TXT wildcard also answers queries for names such as `_dmarc.sub.example.com` if no explicit record exists there. This is usually harmless, but it can complicate troubleshooting because every TXT query receives an answer.

### SPF flattening

Tools that resolve all `include` references and replace them with the underlying IP addresses solve the lookup limit at the cost of maintainability. If the provider changes its addresses, sending fails and nobody notices because everything appears correct in the local record. Anyone taking this approach therefore needs automated reconciliation that regularly checks the list against the source. As one-off manual work, this method will fail sooner or later.

## DKIM

DKIM signs outgoing messages. The public key is located under `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

The selector can be chosen freely and is specified by the sending system. A descriptive name with a date makes later rotation much easier than `s1` and `s2`.

### Delegation via CNAME

Where the sending service offers it, the CNAME variant is preferable to entering the record directly:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

The provider can then rotate its key independently without anyone needing to work in your zone. Otherwise, this rotation is regularly neglected because it requires coordination between two teams. However, a CNAME excludes every other record at the same name; this is a fundamental DNS rule, not a DKIM peculiarity.

### Rotation without downtime

When changing keys, publish the new selector first, then switch the sending server to it, and only then remove the old record. Anyone deleting the old key immediately invalidates the signatures of all messages still in transit or in queues and makes subsequent checks impossible. A lead time of a few days between switching and deletion is appropriate.

A record with an empty `p=` is not a broken entry, by the way, but the specified way to mark a key as revoked.

### Key length

1024 bits are considered obsolete; 2048 bits are standard. Larger RSA keys provide virtually no additional benefit and only increase the chance that an intermediary system will not process the record correctly.

## DMARC

DMARC combines SPF and DKIM with an instruction on what should happen when a check fails, and sends reports back. The record is located under `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Since May 2026, RFC 9989 and the reporting specifications RFC 9990 and RFC 9991 have introduced the revised version, replacing RFC 7489. Three changes matter in practice:

- **`pct` has been removed.** Gradual rollout by percentage no longer exists. It is replaced by `t=y`, which marks the domain as being in testing: reports continue, but the policy should not be enforced.
- **`np` is new.** It sets the policy for non-existent subdomains, closing a gap that attackers like to exploit because invented subdomains were previously covered only by `sp`. Without an explicit setting, `np` follows the value of `sp`.
- **The Public Suffix List has been replaced by a `Tree Walk`.** The organisational domain is no longer determined from an externally maintained list, but through tiered DNS queries along the name tree. For large namespaces with many levels, this noticeably changes evaluation.

### Alignment is the real core

DMARC does not pass merely because SPF or DKIM technically passed, but only when at least one of them also matches the visible sender domain in the `From` header. SPF is checked against the envelope sender domain, which regularly differs for forwarding, newsletter services and ticketing systems. This is precisely why messages with valid SPF sometimes still fail the DMARC check.

With `adkim=r` and `aspf=r` (relaxed, the default), matching at the organisational-domain level is sufficient. `s` requires exact equality including the subdomain and in practice almost always fails on one of the sending paths.

### External report addresses require authorisation

If reports are to go to an address outside your own domain, for example to a DMARC analysis service, the receiving domain must authorise it. Without this record, many recipients simply send nothing, leaving analysis empty while everything looks correct in your own record:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

This entry is created by the operator of the target zone, not by you. With commercial services this happens automatically, but not for a self-operated collection mailbox in another domain you own.

### Typical syntax errors

Tag names and policy values must be lowercase; `p=Reject` is invalid. A semicolon separates tags; a missing separator makes the rest of the line ineffective. And `p` must be the first tag after `v`. A record consisting only of `v=DMARC1; rua=...` contains no policy and is incomplete.

### The rollout

`p=none` is a measurement state, not a goal. It does not change how recipients handle your mail and serves only to identify all legitimate sending paths through reports. Anyone who does not move from `quarantine` to `reject` within a few months after implementation has put in the effort without gaining the protection. The organisational side of this path, including a decision template, is a separate topic and is described in the DMARC blueprint.

## MTA-STS and TLS-RPT

SMTP encrypts opportunistically: if the remote end offers STARTTLS, encryption is used; otherwise, it is not. An attacker in a position to manipulate traffic can remove the STARTTLS announcement and thereby keep the connection in plaintext. MTA-STS closes this gap for receiving domains.

MTA-STS consists of two parts, only one of which is in DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

The actual policy is a file located at `https://mta-sts.example.com/.well-known/mta-sts.txt` and must be served with a valid certificate:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Almost all pitfalls lie outside the zone:

- **The `id` must change with every policy change.** It is the only indication for sending systems that a new policy needs to be retrieved. Anyone changing the file while leaving the `id` unchanged works against cached copies until `max_age` expires.
- **The MX list in the policy and the MX records must match.** A new MX missing from the policy is rejected by senders using `mode: enforce`. During migrations, the policy must therefore be adjusted before switching MX records.
- **`mode: testing` first.** In this mode, violations are only reported, not enforced. Switch to `enforce` when reports are clean.
- **A CAA record can block certificate issuance for the policy host** if it specifies a certificate authority other than the one being used.

TLS-RPT provides the corresponding reports and is a single record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT is useful even without MTA-STS because it makes failed transport encryption visible in the first place.

## DANE

DANE achieves the same goal as MTA-STS but anchors trust in DNS rather than the web PKI. It requires a zone signed end-to-end with DNSSEC, and without DNSSEC a TLSA record is ineffective.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Operationally crucial: the TLSA record must be correct before every certificate change. The usual procedure publishes the new hash alongside the old one, then changes the certificate and finally removes the old entry. Anyone reversing this order makes the mail server unreachable for all DANE-validating senders, including the major German-speaking providers. DANE is much less common in Switzerland than MTA-STS, usually because the zone lacks DNSSEC signing.

## BIMI

BIMI displays the brand logo in the inbox and is the only mechanism discussed here that is not yet an RFC, but is still maintained as an Internet Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

The requirements are high: an enforced DMARC policy with `quarantine` or `reject`, a logo in SVG Tiny Portable/Secure format and, for most providers, a paid Verified Mark Certificate. BIMI is therefore not a security mechanism but a visibility matter, and belongs at the end of the sequence, not the beginning.

## Other records in the surrounding environment

**Autodiscover and SRV:** Exchange environments use `autodiscover.example.com` as a CNAME or an SRV record `_autodiscover._tcp.example.com`. Both concern client configuration rather than mail flow, but are often overlooked during migration and then result in profiles that can no longer be set up.

**CAA:** It is not directly related to mail, but determines which certificate authority may issue a certificate for `mta-sts.example.com` or the mail server name.

**Split-horizon zones:** Where an internal DNS zone has the same name as the public one, mail records often do not exist internally. Internal systems performing SPF or DKIM checks then arrive at different results than the outside world. Every change to mail records should therefore include the question of whether the internal zone needs updating.

## Some short tests

Deliberately send all queries to a public resolver so that neither the internal cache nor a split-horizon zone responds:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Against the authoritative server to bypass caching entirely:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

On Windows without `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

For a complete evaluation including SPF lookup counting, DKIM selector lookup and alignment checking, this page provides the [Mail DNS Check](https://rafaelpfister.ch/tools/mail-check), which checks a domain against all records described here in one pass.

However, the most meaningful test remains a real message. Send an email to a mailbox at a major provider and inspect the `Authentication-Results` line in the header. It shows in one line what SPF, DKIM and DMARC actually produced, replacing every theory about the zone file.

## Order during a migration

When changing mail providers, this sequence has proven effective:

1. Lower the TTL of all affected records to 300 seconds at least one day in advance.
2. Publish the new provider’s DKIM selectors while the old ones are still in place.
3. Extend SPF with the new provider without removing the old one, and recalculate the lookup limit.
4. For MTA-STS, adjust the policy to the new MX names and increment `id` before switching the MX records.
5. Switch MX records and monitor delivery.
6. Only after a few days without issues, remove the old SPF includes and DKIM selectors.
7. Reset the TTL.

The most common issue in this process is doing step 6 too early: old entries are deleted together with the switch, and everything still using the previous route fails sender authentication.

## Conclusion

Mail records differ from all other DNS entries in that an error does not necessarily become obvious. An incorrect A record causes a ticket within minutes, while a duplicate SPF record or a DKIM key with one character too many leads to a delivery rate that slowly declines over weeks.

Three rules prevent most of these cases. First: before creating every new record, check what already exists instead of placing a second one alongside it. Second: after every change, verify against a public resolver and compare the value character by character with the template, not merely visually. Third: during changes, always publish the new configuration first, then switch, then remove the old one. Those who follow this order always have a way back with mail records.

## Kilder

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Defines, among other things, the 255-byte limit of an individual `character-string` in TXT records.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): States in section 10.3 that the target of an MX record must not be an alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Lookup limit of ten mechanisms, void lookup limit, deprecation of the SPF RR type and recommendation against the `ptr` mechanism.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Structure of the key record under `_domainkey`, the significance of the selector and the empty `p=`.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Current DMARC specification from May 2026, replacing RFC 7489; removal of `pct`, new `np` tag, Tree Walk instead of the Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format and delivery of aggregate reports, including authorisation of external recipient domains.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identification of domains that do not accept mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): DNS record, policy file, significance of `id` and the modes `testing` and `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Structure of the `_smtp._tls` record and reports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): TLSA records for SMTP and the requirement for a DNSSEC-signed zone.

11.  [Brand Indicators for Message Identification (BIMI), Internet Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Current status of the BIMI specification; still not an RFC.

12.  [Google: Email sender guidelines](https://support.google.com/a/answer/81126): Requirements for senders, including the PTR requirement for sending IPv6 addresses and the requirements for bulk senders in force since February 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requirements for senders of 5,000 messages per day or more, effective since May 2025.
