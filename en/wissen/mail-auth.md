---
title: "SPF, DKIM, and DMARC: sender authentication in combination"
blatt: "mail-auth"
description: "The three sender authentication mechanisms and how they work together: what SPF checks, what DKIM signs, how DMARC ties both to the visible sender, and the rollout path from reports to p=reject."
fakten:
  - { label: "Components", wert: "SPF (sender IPs), DKIM (signature), DMARC (policy)" }
  - { label: "SPF", wert: "RFC 7208, TXT record of the domain", href: "https://datatracker.ietf.org/doc/html/rfc7208" }
  - { label: "DKIM", wert: "RFC 6376, key published under selector._domainkey", href: "https://datatracker.ietf.org/doc/html/rfc6376" }
  - { label: "DMARC", wert: "RFC 7489, policy published under _dmarc", href: "https://datatracker.ietf.org/doc/html/rfc7489" }
  - { label: "Core concept", wert: "Alignment: the checked domain must match the header From" }
  - { label: "Rollout", wert: "observe with p=none, then quarantine, then reject" }
  - { label: "Foundation", wert: "Everything lives in DNS", href: "/en/kb/dns" }
werbung: ["tools", "newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# SPF, DKIM, and DMARC: sender authentication in combination

Whether a message arrives in the inbox, in the spam folder, or not at all is today decided largely by sender authentication. The major receivers now require it explicitly; bulk mail sent without correct records is throttled or rejected. Each of the three mechanisms is quickly explained on its own, but their value only emerges from the combination.

## What each mechanism checks

**SPF** answers one question: is this server allowed to send on behalf of the domain? A TXT record lists the legitimate sources, and the check runs against the envelope sender. SPF does break on forwarding, however, because delivery then comes from a third-party server with an unrelated IP address.

**DKIM** attaches a cryptographic signature to every message; the public key is published under a selector in DNS. The signature survives forwarding and proves that the message is unchanged since it was sent and that it genuinely belongs to the signing domain.

**DMARC** turns both into an enforceable policy and closes the gap attackers otherwise exploit: SPF and DKIM check technical domains that recipients never see. DMARC requires **alignment**, meaning agreement between the checked domain and the visible header From, and it defines what happens on failure: nothing (`p=none`), quarantine, or rejection (`p=reject`).

## The rollout path

The safe path to `reject` runs through data rather than nerve. First, set up SPF and DKIM for every legitimate sending path, including the newsletter service, the ticketing system, and multifunction devices. Second, publish DMARC with `p=none` and a reporting address (`rua=`) and evaluate the **aggregate reports** for a few weeks; they show in black and white everyone who sends on behalf of the domain. Third, bring forgotten sources into line or shut them down, then move to `quarantine`, then to `reject`, ideally in stages using `pct=`. Cutting the sequence short costs exactly the messages nobody thought of.

## Operational notes

Three recurring items keep the system healthy: **key rotation** for DKIM by way of new selectors (the old selector stays in place until the switch is complete), an **SPF diet** (ten DNS lookups at most, and every provider include counts toward it), and **subdomains**, which must not be forgotten, because an unaddressed subdomain is the open window next to the secured door (the `sp=` policy). For verification, `dig` and a DNS-based mail authentication check are sufficient.

