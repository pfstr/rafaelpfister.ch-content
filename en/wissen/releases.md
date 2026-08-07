---
title: "Releases and Updates: Patch Operations for Mail Infrastructure"
blatt: "releases"
description: "How update operations for Exchange, appliances and gateways become plannable: release types from security update to firmware, maintenance windows and rollback paths, reading release notes, and the question of how quickly patching has to happen."
fakten:
  - { label: "Purpose", wert: "Keeping the security and feature level of the mail infrastructure current" }
  - { label: "Exchange on-prem", wert: "Security updates monthly, cumulative updates twice a year", href: "https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates" }
  - { label: "Appliances", wert: "Firmware releases (SEPPmail, Totemomail, Cisco AsyncOS, HIN)" }
  - { label: "Patch trigger", wert: "Patch day, out of band for actively exploited vulnerabilities" }
  - { label: "CVE reference", wert: "cve.org and vendor advisories", href: "https://www.cve.org/" }
  - { label: "Key questions", wert: "What changes, what breaks, how to get back?" }
werbung: ["newsletter"]
ctaThemen: ["exchange-onprem-hybrid", "seppmail"]
---

# Releases and Updates: Patch Operations for Mail Infrastructure

Mail systems are reachable from the internet, process foreign input and carry professional secrets. There is hardly any infrastructure for which prompt patching matters more, and hardly any for which unplanned outages hurt faster. Good update operations are therefore not an act of heroism on the evening of patch day but a practiced process with known answers to three questions: what changes, what can break, and how to get back?

## Knowing the release types

Not every update is alike. **Security updates** close vulnerabilities and are meant to go through quickly; ideally they change no behavior. **Cumulative updates and firmware releases** bring features, change default values and carry their own prerequisites, such as schema updates in Exchange or new minimum versions of runtime components. **Patch releases** from appliance vendors (for example a 15.0.6.1 after a 15.0.6) usually correct specific defects of the predecessor. Knowing the type means knowing the risk: a security update needs a short window and monitoring, a large release needs a test environment, release notes and a fallback plan.

## Reading release notes as an operator

Release notes answer four operator questions: **fixed vulnerabilities** (with CVE numbers, from which the urgency follows), **behavioral changes** (new defaults, removed functions, changed requirements for neighboring systems), **known issues** (the section that has to be read before the update, not after) and the **update path** (from which versions, in which order, which cluster node first). On gateways it is worth looking at changes to LDAP integration, certificate handling and TLS defaults, because that is where side effects in the mail flow originate.

## The process that holds up

A simple four-step approach has proven itself. **Inventory**: which systems, which versions, which support windows (end of support is a migration topic, not a patching topic). **Assessment**: weigh CVSS and exploitability against the exposure of the system; an externally reachable gateway is patched differently from an internal tool. **Window and sequence**: cluster systems node by node, standby first, then active; configuration backup and snapshot beforehand where possible. **Verification**: after the update, check mail flow in both directions, LDAP bind, certificates and the administration interface before the window closes.

## Fast, but not blind

The two mistakes sit at the extremes: installing immediately and unverified, or doing nothing for months. The workable answer is in between, and it follows from two variables: how exposed the system is, and how easily the step can be undone. An externally reachable gateway with an actively exploited vulnerability justifies a short window; an internal system with a clean snapshot can wait until the first field reports arrive. Either way: snapshot or configuration export before the update, a checkpoint afterwards, and a note recording what was installed.
