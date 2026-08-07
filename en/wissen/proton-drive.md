---
title: "Proton Drive: End-to-End Encrypted Cloud Storage"
blatt: "proton-drive"
description: "An overview of Proton's cloud storage: end-to-end encryption as its basic principle, origin and ecosystem, clients and platforms, and what a zero-knowledge architecture means for third-party tools and automation."
fakten:
  - { label: "Provider", wert: "Proton AG, Geneva", href: "https://proton.me/drive" }
  - { label: "Basic principle", wert: "End-to-end encryption, zero knowledge" }
  - { label: "Ecosystem", wert: "Proton Mail, Calendar, VPN, Pass, Drive" }
  - { label: "Cryptography", wert: "OpenPGP-based, keys held by the user" }
  - { label: "Clients", wert: "Web, Windows, macOS, Android, iOS" }
  - { label: "Jurisdiction", wert: "Switzerland" }
werbung: ["newsletter"]
ctaThemen: ["proton-drive"]
---

# Proton Drive: End-to-End Encrypted Cloud Storage

Proton Drive is the cloud storage service of the Geneva-based provider Proton, which became widely known through Proton Mail. The product is positioned around a single characteristic that shapes its architecture throughout: **end-to-end encryption**. Files, file names and folder structures are encrypted on the user's device; the provider stores only data that it cannot read itself.

## Zero knowledge as architecture

The encryption builds on OpenPGP mechanisms; the keys are tied to the user account and protected by its password. From this follow the properties typical of zero-knowledge services, both the advantages and the limitations: the provider can neither search the content nor hand it over nor use it for previews and server-side processing. Sharing works through cryptographically secured links, and account recovery is possible only through recovery methods set up in advance, because a password reset without a recovery key means lost data. The security model and the clients are documented as open source and have been audited externally.

## Ecosystem and jurisdiction

Drive is part of the Proton account and shares identity, subscription and key material with Mail, Calendar, VPN and the password manager. The company's Swiss domicile places the data outside EU and US cloud legislation, which is a decisive selection criterion for user groups with elevated protection requirements; Proton operates data centers in Switzerland and Europe.

## Clients and platforms

Full functionality is provided by the web client and the apps for Windows, macOS, Android and iOS, including file synchronization and photo backup on mobile devices. **Linux** has historically been the laggard of the client lineup: for a long time no official sync client existed, and the zero-knowledge architecture at the same time prevents third-party tools from simply connecting to the service, because without a published, stable API and client-side cryptography, external tools are either denied access or limited to unofficial, fragile routes. The state of Linux support is therefore a moving target that shifts with Proton's client roadmap.

## Assessment

Proton Drive competes less on storage price or depth of integration than on its trust model: maximum confidentiality in exchange for reduced convenience. Server-side full-text search, broad third-party integration or scriptable access are found in conventional services; for content that is to remain closed even to the provider, this is one of the most consistent offerings available.
