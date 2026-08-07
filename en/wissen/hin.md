---
title: "HIN: secure email in the Swiss healthcare sector"
blatt: "hin"
description: "The HIN platform from an operator's perspective: HIN identities and protected mail between healthcare professionals, the HIN Mail Gateway inside an organization's own infrastructure, Access Gateway and client, platform upgrades and their deadlines."
fakten:
  - { label: "Platform", wert: "HIN (Health Info Net)" }
  - { label: "Operator", wert: "HIN AG, Switzerland", href: "https://www.hin.ch/" }
  - { label: "Purpose", wert: "Secure identities and protected email in healthcare" }
  - { label: "Core", wert: "HIN identity (electronic identity) plus HIN Mail" }
  - { label: "Integration", wert: "HIN Client, Access Gateway, HIN Mail Gateway" }
  - { label: "Gateway connection", wert: "SMTP in the mail path of the organization's own infrastructure" }
  - { label: "Support", wert: "HIN support and documentation", href: "https://support.hin.ch/" }
werbung: ["stargate", "newsletter"]
ctaThemen: ["hin-gateway"]
---

# HIN: secure email in the Swiss healthcare sector

In the Swiss healthcare sector, HIN is the de facto standard for protected electronic communication: physicians, hospitals, pharmacies, laboratories, and insurers exchange patient information over HIN-secured mail. For the IT staff responsible in these organizations, HIN is not an app but infrastructure, with its own on-premises components, its own deadlines, and a mail flow that differs fundamentally from open internet mail traffic.

## Identity first

The foundation of the platform is the **HIN identity**: a verified electronic identity per healthcare professional or organization. The services build on it, above all HIN Mail: messages between HIN participants travel encrypted within the platform, and the identity of the other party is verified. Mail to recipients outside the HIN world is secured through protected delivery paths, for example portal retrieval. User access runs conventionally through the **HIN Client** on the workstation or organization-wide through central components.

## The HIN Mail Gateway in an organization's own mail flow

Organizations with their own mail infrastructure connect to HIN through the **HIN Mail Gateway**: a component in the organization's own data center that sits as an SMTP station between the internal mail system (typically Exchange) and the HIN platform. Internally, users write ordinary mail; the gateway handles encryption and decryption toward the HIN world. The same operating rules therefore apply to the gateway as to any secure mail gateway: it is operationally critical in the mail path, its **configuration and key material belong in the backup**, its releases need to be installed in a planned manner, and its interaction with connectors in the mail system must be documented. In addition, the **Access Gateway** serves as the central access point for user authentication in larger environments.

## Evolution of the platform

HIN develops the platform in generations. Such platform upgrades typically involve new network endpoints, minimum versions for the HIN Client and Access Gateway, and hardware generation changes for the Mail Gateway; they are announced by HIN and accompanied by transition periods.

## Operational view

Three points govern operations: the **delivery paths**, meaning which messages travel through HIN and which through normal mail flow, along with the rules that make that decision; the **version levels** of gateway and clients, because platform upgrades come with minimum versions; and the **identities**, whose mapping to people and roles has to be maintained so that deputies and departures are represented correctly.
