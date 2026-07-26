---
title: "Midea PortaSplit in Home Assistant: Why Token and Key Are Critical"
navTitle: "PortaSplit & Token"
description: "Local control needs two values from the Midea cloud. Here's how token and key are obtained, why losing them is a problem, and how owners can secure their existing setup."
date: "2026-07-24"
kategorie: "Smart Home & IoT"
timeToRead: "9 min to read"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-setup-and-hardening"
  - "serverless-newsletter-cloudflare-workers-d1"
translationOf: "midea-portasplit-home-assistant"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-home-assistant-integration"
url: "https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-integration"
---

<aside class="article-update">
  <p class="article-update__label">What PortaSplit owners should do now</p>
  <p>Home Assistant obtains the device-specific token and key from private cloud interfaces during setup. The Midea AC LAN project has been warning of possible changes since 19 May 2025. However, no concrete shutdown date from the manufacturer is documented. For owners this means:</p>
  <ol>
    <li><strong>Don't break an existing setup without reason.</strong> Only obtaining the credentials needs the Midea cloud. Future changes to the private endpoint could make a fresh setup harder.</li>
    <li><strong>Back up token, key, and configuration in encrypted form.</strong> If retrieval stops working later on, the backup remains the most reliable path to recovery.</li>
    <li><strong>Do not break the pairing without need.</strong> A factory reset, removing the device from the Midea account, or swapping the Wi-Fi module forces a new token retrieval that may fail in the future.</li>
  </ol>
  <p>Devices that are already set up continue to be controlled locally. Changes to the cloud interface therefore primarily affect adding and recovering devices, not every ongoing control command. The concrete steps are in the <a href="/en/blog/midea-portasplit-home-assistant-setup-and-hardening">practical guide on setup and hardening</a>.</p>
</aside>

![Example Home Assistant dashboard for a Midea PortaSplit showing room and target temperature, humidity, power draw, energy consumption, and compressor runtimes over the past 24 hours.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Local control of the Midea PortaSplit rests on two device-specific values: a token and a key. The Home Assistant integration retrieves both during setup through a private Midea cloud endpoint. After that, it sends control commands directly on the local network.

The Midea AC LAN project warns of possible changes to these cloud interfaces. More recent analysis shows, however, that this cannot be read as a confirmed manufacturer roadmap or a concrete shutdown date. This post explains the technical dependency; the [detailed API analysis](/en/blog/midea-v2-cloud-api-clarified-portasplit-home-assistant) untangles the various "V2" labels and the current state of affairs.

## The token question in detail

### Why could Home Assistant obtain the token at all?

The interesting part: the community never computed the token. Instead, it analysed the network traffic of the official app and found that the app does not generate the token itself but fetches it from the cloud:

```text
App
   ↓
Midea cloud
   ↓
cloud returns token
   ↓
app uses token locally
```

The Home Assistant integration reimplemented exactly this cloud call. It signs in to the cloud with the same endpoints and the same flow as the app, and thereby receives the same token and key. The foundation is therefore not a clever calculation but a reconstructed retrieval. If the endpoint disappears, so does the retrieval.

### Could you read the token from the official app?

In theory, yes. The app must know the token at some point, otherwise it could not talk to the device locally. Conceivable routes would be:

- reverse engineering the app,
- sniffing the network traffic, if it is not additionally protected,
- instrumenting the app at runtime, for example with Frida or Objection,
- hooking the functions that process the token.

This is exactly what the Midea AC LAN developer means when saying the previous design is a security problem from Midea's point of view: a long-lived secret that can be extracted from a widely distributed app with reasonable effort is hard to control. For an individual user, though, these routes are laborious and do not replace the convenient cloud retrieval.

### Could you get the token directly from the device?

That would be the most elegant solution. If the device exchanged a public key during the first local pairing, or used a one-time pairing code over Bluetooth, no cloud would be needed at all. Many modern IoT devices do exactly that.

Midea, however, designed the original LAN protocol differently: the device only accepts local commands with the matching, cloud-derived credentials. There is no documented local pairing mechanism that would hand out the token without going through the cloud. The cloud is therefore not just convenience but, architecturally, the only intended path to the token.

### Could the community work around changes to the token endpoint?

That would only be possible if one of the following turns up:

- a new cloud API that still hands out tokens,
- a previously unknown local pairing method,
- a vulnerability in the device,
- or Midea itself publishing an official local API at some point.

Simply "recomputing" the token, by contrast, is very unlikely to work. If it were possible, the community would probably have done it long ago and would never have depended on the cloud API in the first place. That the detour through the cloud was built at all is the strongest indication that there is no simpler local path.

## The Midea AC LAN warning

The `Midea AC LAN` repository carries a prominently placed "Important Notice". According to the developer, Midea has already closed the server-side token APIs in the Meiju cloud and the SmartHome cloud. The integration therefore currently falls back to the token interfaces of the NetHome Plus cloud, and those are expected to be closed progressively too. The consequence would be that already configured devices keep working locally, while new devices can no longer be added. The developer goes further, writing that Midea intends to move to a new cloud control API in the long run and thereby render the existing V1 LAN API unusable.

The warning has a short history. The prominent "Important Notice" was added to the README on 19 May 2025 (pull request #578) and then named the SmartHome cloud as the fallback for adding new devices. On 14 July 2025 (#639) it was updated; since then it points to the NetHome Plus cloud, because Midea had closed further endpoints. The core stayed the same across both versions: the token interfaces are disappearing one after another, only the cloud still usable at the time changes.

This needs a differentiated reading. It is the assessment of an open-source project, not a binding roadmap from Midea, and the timeline is unknown. A future firmware update may change local functionality, and a stored token may keep working, but not necessarily forever. A factory reset, a Wi-Fi module swap, or a new device may require obtaining a token again.

This is where the three steps from the box at the top come from, each with its rationale:

- **Don't replace a working setup without reason.** Token retrieval is the only step that necessarily runs through the Midea cloud. Changes to the private endpoint can mainly affect a later fresh setup.
- **Back up your credentials.** Home Assistant stores token and key locally. A broken system, a failed restore, or an accidentally deleted integration can still render local control unusable if no external backup exists.
- **Don't break the pairing lightly.** Whether a factory reset or removal from the Midea account forces new credentials on every model is not fully documented. A backup before such changes is therefore essential.

Day-to-day operation is not affected for now: local control uses the values already stored and no longer needs the token endpoint. A residual risk remains if a later firmware update changes the local protocol or authentication. How to back up token, key, and configuration in practice is covered in the [practical guide on setup](/en/blog/midea-portasplit-home-assistant-setup-and-hardening#backing-up-the-configuration).

## What this means for security

Beyond availability, the warning also has a security core. According to `Midea AC LAN`, the older LAN architecture rests on a problematic assumption: client communication was originally considered sufficiently protected, which is why the tokens issued by the cloud were given no expiry.

A non-expiring token is not a vulnerability by itself. It becomes a problem when it ends up in logs or unprotected backups, reaches third parties, or can neither be revoked nor rotated. The developer of `Midea AC LAN` suspects that Midea is responding to these risks with changes to the token services and a more cloud-based architecture. A corresponding manufacturer announcement with a timeline, however, is not substantiated.

Precision of language matters here. The community integration does not "hack" the air conditioner. It implements a proprietary protocol that was reconstructed through reverse engineering. The security problem arises from long-lived secrets being usable and storable outside the app they were originally intended for.

For operation on your own network, what matters most is what token and key enable. Both authenticate the local communication with the device. If they fall into the wrong hands, an attacker could, depending on the protocol and their network position, discover the device, authenticate against it, read status information, change settings, switch the air conditioner on or off, change operating modes, and change the target temperature. To do so, the attacker still generally needs a network connection to the device; possessing token and key alone does not enable an attack from the open internet. Token and key are therefore to be treated like a password. How to embed the device on the network so that these values do little harm even after a slip-up is the subject of the [second part](/en/blog/midea-portasplit-home-assistant-setup-and-hardening#operating-the-portasplit-securely).

## What Practically Remains

Local control of the PortaSplit stands and falls with a token and key that currently can only be obtained through the Midea cloud. This detour is part of the protocol design: local commands are tied to cloud-derived credentials. Because the endpoint is private and undocumented, the long-term availability of the unofficial integration remains uncertain.

In practice, that means: back up credentials and configuration, don't break a working pairing without need, and keep an eye on changes to the integration and firmware. Devices already set up keep running locally. Setup, backup, and network hardening are covered in the [practical guide to the PortaSplit](/en/blog/midea-portasplit-home-assistant-setup-and-hardening).

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: The `Midea AC LAN` integration with the "Important Notice" (since 19 May 2025, updated 14 July 2025), the rationale around non-expiring tokens and reconstructed client encryption, and the description of the cloud-based token retrieval.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: The `Midea Smart AC` integration: description of the cloud-based token and key retrieval on V3 devices and the local storage of the values.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Vendor information on the SmartHome ecosystem and the referenced security and privacy standards.
