---
title: "DNS for mail admins: MX, SPF, DKIM, and DMARC in one place"
blatt: "dns"
description: "Why email does not work without DNS: MX and PTR records, SPF, DKIM, and DMARC as TXT records, TTL and propagation, and how to verify entries properly before mail flow suffers."
fakten:
  - { label: "Full name", wert: "Domain Name System" }
  - { label: "Purpose", wert: "Resolve names to addresses and publish policies for mail" }
  - { label: "Introduced", wert: "1983 · RFC 1034/1035" }
  - { label: "OSI layer", wert: "Application (layer 7)" }
  - { label: "Transport", wert: "UDP and TCP, port 53", href: "https://datatracker.ietf.org/doc/html/rfc1035" }
  - { label: "Mail records", wert: "MX, SPF (TXT), DKIM (TXT), DMARC (TXT), PTR" }
  - { label: "SPF", wert: "RFC 7208", href: "https://datatracker.ietf.org/doc/html/rfc7208" }
  - { label: "DKIM", wert: "RFC 6376", href: "https://datatracker.ietf.org/doc/html/rfc6376" }
  - { label: "DMARC", wert: "RFC 7489", href: "https://datatracker.ietf.org/doc/html/rfc7489" }
  - { label: "Tools", wert: "dig, nslookup, delv" }
werbung: ["tools", "newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# DNS for mail admins: MX, SPF, DKIM, and DMARC in one place

Email is the most DNS-dependent system in an organization. Before a single byte of a message travels, DNS queries decide where it goes, and after delivery further DNS records decide whether it lands in the inbox or in the spam folder. Operating mail flow always means operating DNS as well, intentionally or not.

## The delivery records

The **MX record** names a domain's mail servers together with a priority; the sending server picks the entry with the lowest value and falls back to the next ones when a host is unreachable. Behind every MX name there must be an **A or AAAA record**; a CNAME is not permitted at this position. On the return path, the **PTR record** (reverse DNS) matters: large providers reject servers whose IP address does not resolve back to a matching hostname. MX, A, and PTR form the minimum that every sending infrastructure has to set up correctly.

## The reputation records: SPF, DKIM, DMARC

**SPF** is a TXT record that lists which systems are allowed to send on behalf of the domain. It is checked against the envelope sender. The most common errors are multiple SPF records (not permitted, there must be exactly one), more than ten DNS lookups within the record, and a forgotten service provider whose mail suddenly ends up on `softfail`.

**DKIM** signs outbound mail cryptographically; the public key is published as a TXT record under a selector (`selector._domainkey.example.ch`). Recipients verify the signature and therefore know that the content and the sender domain belong together. Selectors allow key rollover without interruption: publish the new key under a new selector, switch sending over, and remove the old selector later.

**DMARC** ties both to the visible header From address and turns them into a policy: `p=none` only observes, while `p=quarantine` and `p=reject` enforce. DMARC reports provide the data on which systems actually send on behalf of the domain. The path to `reject` always leads through an inventory based on these reports; enforcing the policy blindly first breaks the forgotten newsletter tools and multifunction devices.

## TTL, propagation, and the time factor

Every record carries a **TTL** that determines how long resolvers cache the answer. Before planned changes such as a server replacement or a gateway migration, it is worth lowering the TTL days in advance and raising it again after the switch. "DNS propagation" is not fate but cache arithmetic: a change takes effect everywhere once the old TTL has expired at the latest.

## Verify instead of guess

```bash
dig MX example.ch +short
dig TXT example.ch +short
dig TXT _dmarc.example.ch +short
dig TXT selector1._domainkey.example.ch +short
dig -x 203.0.113.25 +short
```

