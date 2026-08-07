---
title: "Migration and deadlines: keeping mail infrastructure life cycles under control"
blatt: "migration"
description: "Why migrations in the mail environment are almost always driven by deadlines: end-of-support dates, vendor platform renewals, migration patterns from big bang to coexistence, and the checklist for cutover day."
fakten:
  - { label: "Purpose", wert: "Orderly system replacement before support and operational deadlines expire" }
  - { label: "Drivers", wert: "End of support, platform renewals, license and contract changes" }
  - { label: "Microsoft reference", wert: "Product lifecycle with fixed end-of-support dates", href: "https://learn.microsoft.com/en-us/lifecycle/" }
  - { label: "Patterns", wert: "Big bang, coexistence, parallel operation with a pilot group" }
  - { label: "Critical items", wert: "Firewall rules, DNS changes, certificates, client versions" }
  - { label: "Rule of thumb", wert: "The vendor's deadline is the latest possible date, not the target date" }
werbung: ["stargate", "newsletter"]
ctaThemen: ["hin-gateway", "exchange-onprem-hybrid"]
---

# Migration and deadlines: keeping mail infrastructure life cycles under control

Hardly any migration in the mail environment begins voluntarily. The trigger is almost always a deadline: an Exchange release drops out of support, a vendor renews its platform and switches off the old one, a contract expires. Managing life cycles actively turns schedule pressure into projects that can be planned. Ignoring them means migrating under time pressure, and in a mail flow, time pressure is the most expensive project partner there is.

## Taking inventory of deadlines

The first step is a plain table: every system in the mail path with its version, end-of-support date, and dependencies. For Microsoft products, the lifecycle database supplies binding dates; appliance vendors communicate platform and firmware deadlines through announcements and portals. Three categories matter: **end of support** (no more security updates, which effectively makes it a security date), **forced platform changes** (the vendor switches over and customers have to follow), and **soft deadlines** (feature deprecations that degrade operations gradually). The project calendar follows from the table on its own, ideally with a buffer of several months ahead of every hard date.

## Choosing the migration pattern

Three patterns cover almost everything. **Big bang**: everything on a single cutover date, appropriate when the systems are small or the vendor permits no coexistence. **Coexistence**: old and new run in parallel and traffic moves over step by step. This is the standard for Exchange migrations and gateway replacements, because the routes back stay open. **Pilot group**: a subset switches first and supplies practical experience before the rest follows. The choice hinges on a single question: what happens if something does not work on cutover day? If the answer is "mail flow stops," the project needs coexistence, or at least a tested route back.

## The technical pitfalls

Migration projects rarely fail because of the target system and often fail at the periphery. Four items belong on every checklist: **firewall rules** (new endpoints and IP addresses must be reachable before the switch, and responsibility for them often sits outside the mail team), **DNS** (MX, Autodiscover, and SPF records, with TTLs lowered in good time), **certificates and keys** (a gateway's domain certificates have to move to the new platform, otherwise older messages stay behind in encrypted form), and **minimum client versions** (platform renewals often require new client or agent versions, and rolling those out takes longer than any server change).

## Cutover day itself

On cutover day, preparation counts for more than speed. What works is a short runbook with clear responsibilities, defined checkpoints after every step and a fallback path that stays valid until a fixed point in time. In practice that means: DNS changes with TTLs shortened beforehand, test messages in both directions immediately after the switch, watching queues and rejections for several hours, and a follow-up phase in which the old path still accepts mail but is no longer used actively.
