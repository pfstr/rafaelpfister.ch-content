---
title: "Midea PortaSplit in Home Assistant: Why token and key are crucial"
navTitle: "PortaSplit & token"
description: "Local control requires two values from the Midea cloud. This is how to obtain the token and key, why losing them is problematic, and how owners can protect their existing setup."
date: "2026-07-24"
kategorie: "Home Assistant and IoT"
timeToRead: "9 min read"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-einrichten
  - serverloser-newsletter-cloudflare-workers-d1
translationOf: "midea-portasplit-home-assistant"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-home-assistant-integration"
url: "https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-integration"
translationId: article-a02e26cce22063f1
translatedAt: 2026-07-28T11:10:30.446Z
translationReview: automatic
translationSourceHash: a02265cf4b8fde907361c3551326fd3283c83d660cf9fdfb40451a9e78ca690b
---

<aside class="article-update">
  <p class="article-update__label">What PortaSplit owners should do now</p>
  <p>Home Assistant retrieves the device-specific token and key during setup via private cloud interfaces. The Midea AC LAN project has warned of possible changes since 19 May 2025. However, no specific manufacturer shutdown date has been documented. For owners, this means:</p>
  <ol>
    <li><strong>Do not unnecessarily dismantle an existing setup.</strong> Only obtaining the credentials requires the Midea cloud. Future changes to the private endpoint could make setting it up again more difficult.</li>
    <li><strong>Back up the token, key and configuration in encrypted form.</strong> If retrieval no longer works later on, the backup remains the most reliable way to restore it.</li>
    <li><strong>Do not remove the pairing without good reason.</strong> Factory resets, removing the device from the Midea account or replacing the Wi-Fi module require obtaining a new token, which may fail in future.</li>
  </ol>
  <p>Devices that have already been set up are controlled locally. Changes to the cloud interface therefore initially affect adding and restoring devices, not every control command in operation. The specific steps are set out in the <a href="/blog/midea-portasplit-home-assistant-einrichten">practical guide to integration and protection</a>.</p>
</aside>

![Example Home Assistant dashboard for a Midea PortaSplit, showing room and target temperature, humidity, power consumption, energy usage and compressor runtimes over the past 24 hours.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Local control of the Midea PortaSplit is based on two device-specific values: token and key. During setup, the Home Assistant integration retrieves both via a private Midea cloud endpoint. It then sends control commands directly over the local network.

The Midea AC LAN project warns of possible changes to these cloud interfaces. More recent analysis shows, however, that no confirmed manufacturer roadmap or specific shutdown date can be inferred from this. This article explains the technical dependency; the [detailed API analysis](/blog/midea-v2-cloud-api-portasplit-home-assistant) puts the various “V2” designations and the current situation into context.

## The token question in detail

### Why was Home Assistant able to obtain the token in the first place?

The interesting part is that the community never calculated the token. Instead, it analysed the official app's network traffic and found that the app does not generate the token itself, but retrieves it from the cloud:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

The Home Assistant integration reimplemented precisely this cloud call. It signs in to the cloud using the same endpoints and process as the app, thereby receiving the same token and key. The actual foundation is therefore not a clever calculation, but a replicated retrieval process. If the endpoint disappears, so does the ability to obtain the credentials.

### Could the token be extracted from the official app?

In theory, yes. The app must know the token at some point, otherwise it could not communicate locally with the device. Conceivable approaches include:

- reverse engineering the app,
- monitoring network traffic, if it is not additionally protected,
- instrumenting the app at runtime, for example with Frida or Objection,
- hooking the functions that process the token.

This is exactly what the Midea AC LAN developer means by saying that the previous design is a security issue from Midea's perspective: a long-lived secret that can be extracted from a widely distributed app with reasonable effort is difficult to control. For individual users, however, these approaches are time-consuming and do not replace the convenience of cloud retrieval.

### Could the token be obtained directly from the device?

That would be the most elegant solution. If the device exchanged a public key during initial local pairing or used a one-time pairing code via Bluetooth, no cloud would be needed at all. Many modern IoT devices work exactly this way.

However, Midea designed the original LAN protocol differently: the device accepts local commands only with the appropriate cloud-related credentials. There is no documented local pairing mechanism that would provide the token without going through the cloud. The cloud is therefore not merely a convenience, but architecturally the only intended route to the token.

### Could the community work around changes to the token endpoint?

That would only be possible if one of the following options is found:

- a new cloud API that continues to provide tokens,
- a previously unknown local pairing method,
- a vulnerability in the device,
- or Midea itself eventually publishes an official local API.

Simply “recalculating” the token, on the other hand, is very unlikely to work. If it were possible, the community would presumably have implemented it long ago and would never have depended on the cloud API. The fact that the cloud detour was built at all is the strongest indication that there is no simpler local route.

## The Midea AC LAN warning

The repository of `Midea AC LAN` contains a prominently placed “Important Notice”. According to the developer, Midea has already closed the server-side token APIs in the Meiju and SmartHome clouds. The integration therefore currently relies on token interfaces from the NetHome Plus cloud, and these too are expected to be gradually closed. The result would be that already configured devices continue to work locally, but new devices can no longer be added. The developer goes further, writing that Midea intends to move to a new Cloud Control API in the long term, rendering the existing V1 LAN API unusable.

The warning has a short history. The prominent “Important Notice” was added to the README on 19 May 2025 (pull request #578) and at the time named the SmartHome cloud as the fallback for adding new devices. It was updated on 14 July 2025 (#639); since then, it has referred to the NetHome Plus cloud because Midea had closed further endpoints. The core message remained unchanged across both versions: token interfaces are gradually disappearing; only the cloud that remains usable at any given time changes.

This needs to be viewed with nuance. It is the assessment of an open-source project, not a binding roadmap from Midea, and the timetable is unknown. A future firmware update may change local functions, and an already stored token may continue to work, but it may not do so forever. A factory reset, a change of Wi-Fi module or a new device may require obtaining a new token.

This leads to the three steps from the box at the start of the article, each with its rationale:

- **Do not replace a working setup without reason.** Obtaining the token is the only step that necessarily goes through the Midea cloud. Changes to the private endpoint may primarily affect a later fresh setup.
- **Back up credentials.** Home Assistant stores the token and key locally. A faulty system, failed restore or accidentally deleted integration can still make local control unusable if no external backup exists.
- **Do not remove the pairing lightly.** Whether a factory reset or removal from the Midea account forces new credentials for every model is not fully documented. A backup before such changes is therefore essential.

Ongoing operation is initially unaffected: local control uses the values already stored and no longer needs the token endpoint. Some residual risk remains if future firmware changes the local protocol or authentication. How to back up the token, key and configuration is explained in the [practical setup guide](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## What this means for security

Besides availability, the warning has a security-related core. According to `Midea AC LAN`, the older LAN architecture is based on a problematic assumption: client communication was originally considered sufficiently protected, which is why the tokens issued by the cloud were given no expiry date.

A token that does not expire is not, in itself, a vulnerability. It becomes problematic if it ends up in logs or unprotected backups, reaches third parties, or can neither be revoked nor rotated. The developer of `Midea AC LAN` suspects that Midea is responding to these risks with changes to token services and a more cloud-based architecture. However, no corresponding manufacturer announcement with a timetable has been substantiated.

Linguistic precision is important here. The community integration does not “hack” the air conditioner. It implements a proprietary protocol that has been understood through reverse engineering. The security issue arises because long-lived secrets can be used and stored outside the originally intended app.

For operation on your own network, what token and key enable is most relevant. Both authenticate local communication with the device. If they fall into the wrong hands, an attacker could, depending on the protocol and their network position, identify the device, authenticate to it, read status information, change settings, turn the air conditioner on or off, switch operating modes and change the target temperature. The attacker would generally still need to establish a network connection to the device; possessing the token and key alone does not enable an attack from across the entire internet. Token and key should therefore be treated like a password. How to integrate the device into the network so that these values cause little harm even in the event of an incident is the subject of the [second part](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## What remains in practice

Local control of the PortaSplit depends entirely on the token and key, which can currently only be obtained through the Midea cloud. This detour is part of the protocol design: local commands are tied to cloud-related credentials. Because the endpoint is private and undocumented, the long-term availability of the unofficial integration remains uncertain.

In practical terms, this means: back up credentials and configuration, do not unnecessarily remove a working pairing, and monitor changes to the integration and firmware. Devices that have already been set up continue to run locally. Setup, backup and network protection are described in the [practical guide to the PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` with the “Important Notice” (since 19 May 2025, updated on 14 July 2025), the explanation concerning non-expiring tokens and reconstructed client encryption, and the description of cloud-based token retrieval.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: Description of cloud-based token and key retrieval for V3 devices and local storage of the values.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Manufacturer information on the SmartHome ecosystem and the referenced security and privacy standards.
