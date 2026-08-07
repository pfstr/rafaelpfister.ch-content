---
title: "Licenses and limits: licensing models for infrastructure software"
blatt: "lizenzierung"
description: "How infrastructure software is licensed: named users, devices, and capacity limits, how license counters are fed technically, what happens when a limit is reached, and the role the directory plays in it."
fakten:
  - { label: "Common models", wert: "Named users, devices/instances, capacity, subscription" }
  - { label: "Counter source", wert: "often the directory (LDAP/Entra) or the product's own user database" }
  - { label: "Behavior at the limit", wert: "hard block, grace period, or notice only, depending on the vendor" }
  - { label: "Metrics", wert: "active accounts, mailboxes, domains, throughput" }
  - { label: "Related", wert: "LDAP as the data source behind the counters", href: "/en/kb/ldap" }
werbung: ["newsletter"]
ctaThemen: ["totemomail"]
---

# Licenses and limits: licensing models for infrastructure software

Licensing models determine more than cost; they also determine operational behavior. Many products count their own usage and change how they behave once a limit is reached. Knowing that mechanism puts messages such as "licensed user limit reached" in perspective: they point to a data problem, not necessarily to a procurement problem.

## The common models

**Named user** licenses cover individual people, independent of concurrent use. They are typical for mail and encryption products, where internal users are counted while external communication partners remain free of charge. **Device or instance licenses** count appliances, servers, or nodes, often tiered by performance class. **Capacity models** tie the price to metrics such as mailboxes, domains, storage, or throughput. **Subscriptions** shift all of these models from a one-time purchase plus maintenance to recurring payment; depending on the product, an expired subscription means loss of functionality, a halt to updates, or merely the end of support.

## Where the counter gets its numbers

Products with user licenses maintain an internal user database, and that database is often populated automatically, typically from the directory over LDAP or from mail traffic itself, where anyone who sends a message gets an entry. This is exactly where the classic discrepancy arises. Alongside active people, the directory also contains departures as well as functional and test accounts, and the license counter draws no distinction based on whether an entry still serves a purpose. A "full" counter therefore often reflects the maintenance state of the directory rather than actual demand. The technical answer is hygiene at the source: precise filters in the integration (which OUs, which groups, which attributes) and regular deactivation of orphaned entries inside the product.

## Behavior at the limit

Vendors implement limits differently: **hard blocks** (new users are rejected, which in a mail flow is a serious operational event), **grace periods** (a time-boxed overrun accompanied by warnings), or **compliance notices only** (usage continues but is settled at audit time). The documentation for the product in question answers two operational questions: what exactly counts as a licensed user, and what happens on the day the limit is reached.

## Verifiability

Every licensing model raises the question of measurability: which report shows the current state, whether it can be exported, and whether the product's counting method matches the contractual definition. Products with a directory integration usually allow both views, the product's own figure and an independent cross-check by LDAP query.
