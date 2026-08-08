---
title: "Midea PortaSplit in Home Assistant: Why Token and Key Matter"
navTitle: "PortaSplit & Token"
description: "Local control requires two values from the Midea cloud. Here is how to obtain the token and key, why losing them is problematic, and how owners can protect their existing setup."
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
translationId: article-a02e26cce22063f1
translatedAt: 2026-08-08T14:14:54.742Z
translationReview: automatic
translationSourceHash: e2e3c42704dc7a3f4618f688790356c5a0ccfa18e0796789bd48cf9841bed1a8
url: https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-integration
translationModel: gpt-5.6-terra
---

<aside class="article-update">
  <p class="article-update__label">What PortaSplit owners should do now</p>
  <p>During setup, Home Assistant obtains the device-specific token and key through private cloud interfaces. The Midea AC LAN project has warned of possible changes since May 19, 2025. However, no specific manufacturer shutdown date has been documented. For owners, this means:</p>
  <ol>
    <li><strong>Do not unnecessarily remove an existing setup.</strong> Only obtaining the credentials requires the Midea cloud. Future changes to the private endpoint could make setting it up again more difficult.</li>
    <li><strong>Back up the token, key, and configuration in encrypted form.</strong> If retrieval no longer works later, the backup remains the most reliable way to restore the setup.</li>
    <li><strong>Do not unpair it without good reason.</strong> Factory resets, removing it from the Midea account, or replacing the Wi-Fi module require obtaining a new token, which may fail in the future.</li>
  </ol>
  <p>Devices that have already been set up are controlled locally. Changes to the cloud interface therefore affect adding and restoring devices first, not every ongoing control command. The specific steps are covered in the <a href="/blog/midea-portasplit-home-assistant-einrichten">practical guide to integration and protection</a>.</p>
</aside>

![Example Home Assistant dashboard for a Midea PortaSplit showing room and target temperature, humidity, power consumption, energy consumption, and compressor run times over the last 24 hours.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Local control of the Midea PortaSplit relies on two device-specific values: the token and key. The Home Assistant integration retrieves both through a private Midea cloud endpoint during setup. It then sends control commands directly over the local network.

The Midea AC LAN project warns of possible changes to these cloud interfaces. However, more recent analyses show that no confirmed manufacturer roadmap or specific shutdown date can be inferred from this. This article explains the technical dependency; the [detailed API analysis](/blog/midea-v2-cloud-api-portasplit-home-assistant) puts the various “V2” labels and the current status into context.

## The token question in detail

### Why was Home Assistant able to obtain the token in the first place?

The interesting part is that the community never calculated the token. Instead, it analyzed the network traffic of the official app and found that the app does not generate the token itself at all, but obtains it from the cloud:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

The Home Assistant integration reimplemented this exact cloud call. It signs in to the cloud using the same endpoints and flow as the app and thus receives the same token and key. The actual foundation is therefore not a clever calculation, but a replicated retrieval process. If the endpoint disappears, so does the ability to obtain them.

### Could the token be extracted from the official app?

In theory, yes. At some point, the app must know the token; otherwise, it could not communicate locally with the device. Generally conceivable approaches would include:

- reverse engineering the app,
- monitoring network traffic, if it is not additionally protected,
- instrumenting the app at runtime, for example with Frida or Objection,
- hooking the functions that process the token.

This is precisely what the Midea AC LAN developer means by saying that the previous design is a security issue from Midea’s perspective: a long-lived secret that can be extracted from a widely distributed app with reasonable effort is difficult to control. For individual users, however, these approaches are demanding and do not replace convenient cloud retrieval.

### Could the token be obtained directly from the device?

That would be the most elegant solution. If the device exchanged a public key during initial local pairing or used a one-time pairing code via Bluetooth, no cloud would be needed at all. Many modern IoT devices do exactly that.

However, Midea designed the original LAN protocol differently: the device accepts local commands only with the appropriate cloud-related credentials. There is no documented local pairing mechanism that would provide the token without going through the cloud. The cloud is therefore not merely a convenience, but architecturally the only intended route to the token.

### Could the community work around changes to the token endpoint?

That would be possible only if one of the following options is found:

- a new cloud API that continues to provide tokens,
- a previously unknown local pairing method,
- a vulnerability in the device,
- or Midea itself eventually publishes an official local API.

Simply “recalculating” the token, by contrast, is very unlikely to work. If it were possible, the community would presumably have implemented it long ago and would never have depended on the cloud API. The fact that the cloud detour was built at all is the strongest indication that no simpler local route exists.

## The warning from Midea AC LAN

The repository of `Midea AC LAN` contains a prominently placed “Important Notice.” According to the developer, Midea has already closed the server-side token APIs in the Meiju and SmartHome clouds. The integration therefore currently uses token interfaces from the NetHome Plus cloud, and these too are supposed to be closed gradually. The consequence would be that devices already set up would continue to work locally, but new devices could no longer be added. The developer goes further, writing that Midea intends to transition to a new Cloud Control API in the long term, making the previous V1 LAN API unusable.

The warning has a brief history. The prominent “Important Notice” was added to the README on May 19, 2025 (Pull Request #578) and at that time named the SmartHome cloud as a fallback for adding new devices. It was updated on July 14, 2025 (#639); since then, it has pointed to the NetHome Plus cloud because Midea had closed additional endpoints. The core remained unchanged in both versions: token interfaces are disappearing one by one, with only the cloud still usable at a given time changing.

This needs to be viewed with nuance. It is the assessment of an open-source project, not a binding roadmap from Midea, and the timeline is unknown. A future firmware update may change local functions, and an already stored token may continue to work, but it may not work forever. A factory reset, a Wi-Fi module change, or a new device may require obtaining a new token.

This leads to the three steps in the box at the beginning of the article, each with its rationale:

- **Do not replace a working setup without reason.** Obtaining the token is the only step that necessarily runs through the Midea cloud. Changes to the private endpoint may primarily affect a later new setup.
- **Back up credentials.** Home Assistant stores the token and key locally. A failed system, an unsuccessful restore, or an accidentally deleted integration can nevertheless make local control unusable if no external backup exists.
- **Do not unpair it carelessly.** Whether a factory reset or removal from the Midea account forces new credentials on every model is not fully documented. A backup before such changes is therefore essential.

Ongoing operation is not initially affected: local control uses the values already stored and no longer needs the token endpoint. A residual risk remains if later firmware changes the local protocol or authentication. How to back up the token, key, and configuration is explained in the [practical setup guide](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## What this means for security

In addition to availability, the warning has a security-related core. According to `Midea AC LAN`, the older LAN architecture is based on a problematic assumption: client communication was originally considered sufficiently protected, which is why the tokens issued by the cloud were not given an expiration date.

A token that does not expire is not, in itself, a vulnerability. It becomes problematic if it ends up in logs or unprotected backups, reaches third parties, or can neither be revoked nor rotated. The developer of `Midea AC LAN` suspects that Midea is responding to these risks with changes to token services and a more cloud-based architecture. However, there is no evidence of a corresponding manufacturer announcement with a timeline.

Linguistic precision matters here. The community integration does not “hack” the air conditioner. It implements a proprietary protocol that was understood through reverse engineering. The security issue arises because long-lived secrets can be used and stored outside the originally intended app.

For operation on your own network, what matters most is what the token and key enable. Both authenticate local communication with the device. If they fall into the wrong hands, an attacker could, depending on the protocol and their network position, identify the device, authenticate to it, read status information, change settings, turn the air conditioner on or off, switch operating modes, and change the target temperature. The attacker generally must still be able to establish a network connection to the device; possession of the token and key alone does not enable an attack from across the entire internet. Token and key should therefore be treated like a password. How to integrate the device into the network so that these values cause little damage even in the event of a mishap is the subject of the [second part](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## What remains in practice

Local control of the PortaSplit depends entirely on the token and key, which can currently be obtained only through the Midea cloud. This detour is part of the protocol design: local commands are tied to cloud-related credentials. Because the endpoint is private and undocumented, the long-term availability of the unofficial integration remains uncertain.

In practice, this means: back up credentials and configuration, do not unnecessarily unpair a working setup, and monitor changes to the integration and firmware. Devices that have already been set up continue to operate locally. The [practical PortaSplit guide](/blog/midea-portasplit-home-assistant-einrichten) describes setup, backup, and network protection.

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` with the “Important Notice” (since May 19, 2025, updated on July 14, 2025), the rationale concerning non-expiring tokens and reconstructed client encryption, and the description of cloud-based token retrieval.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: description of cloud-based token and key retrieval for V3 devices and local storage of the values.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Manufacturer information on the SmartHome ecosystem and the referenced security and privacy standards.
