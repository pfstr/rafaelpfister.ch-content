---
title: "Midea PortaSplit in Home Assistant: Local Control and the Cloud Token Question"
navTitle: "PortaSplit & Token"
description: "The Midea PortaSplit can be controlled locally from Home Assistant, but setup needs a one-time token and key from the Midea cloud. How that token works, why Home Assistant could obtain it, why Midea is shutting the interfaces down, and what owners should do now."
date: "2026-07-24"
kategorie: "Smart Home & IoT"
timeToRead: "10 min to read"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-setup-and-hardening"
  - "serverless-newsletter-cloudflare-workers-d1"
translationOf: "midea-portasplit-home-assistant"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-home-assistant-integration"
url: "https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-integration"
aiPrompt: |
  Explain the token architecture of the local Midea Home Assistant integrations:
  1. Why does local control of a V3 device need a token and key at all?
  2. How does the community integration obtain these values, and why does that run through the Midea cloud?
  3. What does Midea shutting down these cloud token interfaces mean for devices already set up versus new ones?
  4. Why can't the token simply be computed locally?
  Put into context what a stored token and key make possible in practice and how to protect them.
---

<aside class="article-update">
  <p class="article-update__label">The Midea AC LAN project warns: Midea is shutting down the cloud interfaces</p>
  <p>Home Assistant uses these interfaces during setup to obtain the device-specific token and key. The notice has stood in the project repository since 19 May 2025. For PortaSplit owners this means:</p>
  <ol>
    <li><strong>Set it up now.</strong> Only the initial token retrieval needs the Midea cloud. If you wait, you may no longer be able to add the device to Home Assistant.</li>
    <li><strong>Back up token, key, and the configuration encrypted.</strong> After the shutdown these values are unlikely to be obtainable again; the backup is then the only path to a fresh setup.</li>
    <li><strong>Do not break the pairing without need.</strong> A factory reset, removing the device from the Midea account, or swapping the Wi-Fi module forces a new token retrieval that may fail in the future.</li>
  </ol>
  <p>Devices that are already set up continue to be controlled locally; as things stand, the shutdown affects adding devices, not operating them. The background is in this article; the concrete setup and backup steps are in the <a href="/en/blog/midea-portasplit-home-assistant-setup-and-hardening">second part on setup and hardening</a>.</p>
</aside>

![Example Home Assistant dashboard for a Midea PortaSplit showing room and target temperature, humidity, power draw, energy consumption, and compressor runtimes over the past 24 hours.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

The Midea PortaSplit is one of the more interesting portable air conditioners for rented flats: it is built as a split unit, with a slim outdoor element hanging outside the window and the main unit standing in the room, and installation needs no permanent structural changes. Less well known is that the PortaSplit can also be integrated into Home Assistant, which allows it to be controlled based on room temperature, electricity price, window state, or presence.

After setup, control runs largely locally on your own network. It does not come entirely without a cloud, though, and that is exactly the part currently in motion: the Midea AC LAN project warns that Midea is shutting down the cloud interfaces Home Assistant uses to obtain the credentials needed for local communication. This first part explains how those credentials work technically, why Home Assistant could obtain them at all, and what the shutdown means in practice. The step-by-step setup and the network hardening are in the [second part on setup and hardening](/en/blog/midea-portasplit-home-assistant-setup-and-hardening).

One note up front: the integrations described here come from the community and are supported neither by Midea nor by Home Assistant officially. Firmware updates, changes to the Midea cloud, or changes to the integrations themselves can affect their behaviour at any time.

## Short answer

Yes, the Midea PortaSplit can be integrated into Home Assistant. Two community integrations are available:

<div class="repo-cards">
  <a class="repo-card" href="https://github.com/mill1000/midea-ac-py" rel="noopener">
    <span class="repo-card__name"><svg width="15" height="15" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg><span>mill1000/midea-ac-py</span></span>
    <span class="repo-card__desc">Midea Smart AC: focused on air conditioners, supports device types 0xAC and 0xCC and the PortaSplit including Out Silent Mode.</span>
    <span class="repo-card__host">github.com</span>
  </a>
  <a class="repo-card" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener">
    <span class="repo-card__name"><svg width="15" height="15" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg><span>wuwentao/midea_ac_lan</span></span>
    <span class="repo-card__desc">Midea AC LAN: covers numerous other Midea device types besides air conditioners, but carries the warning about the cloud token APIs being shut down.</span>
    <span class="repo-card__host">github.com</span>
  </a>
</div>

Once set up, both communicate with the air conditioner directly over the local network. On newer Midea devices, however, the cloud is needed once to obtain a device-specific token and key. Which of the two integrations fits the PortaSplit better and how setup works is covered in the [second part](/en/blog/midea-portasplit-home-assistant-setup-and-hardening). Here the focus is first on what this token is and why it sits at the centre of the current warning.

## Why local control needs a cloud

This is the key technical difference between "local control" and "a setup entirely free of the cloud". After setup, the actual control commands go directly from Home Assistant to the PortaSplit:

```text
Home Assistant → local network → Midea PortaSplit
```

That means a switching command does not have to travel through an external Midea server, response times are short, an outage of the Midea cloud does not necessarily break an already configured local setup, and the device generally remains controllable without internet access.

On newer devices using the so-called V3 protocol, however, the PortaSplit does not accept local commands unprotected. Home Assistant needs two device-specific values: a token and a key. Both authenticate and encrypt the local connection. Without them you can open a TCP connection to the device but cannot issue a valid command. The integration does not know these values on its own; it retrieves them through a Midea cloud interface during initial setup. The developers of `Midea Smart AC` state this explicitly: for V3 devices, the Midea cloud is used during discovery to obtain token and key. Those are then stored locally, and no cloud connection is required for further control.

Simplified, the sequence looks like this:

1. The PortaSplit is paired with MSmartHome.
2. Home Assistant signs in to a Midea cloud.
3. Home Assistant receives device ID, token, and key.
4. Token and key are stored locally.
5. Home Assistant controls the PortaSplit directly over the LAN.

"Locally controllable" therefore does not automatically mean "never connected to a cloud". The cloud is needed once to obtain the credentials, not for day-to-day operation.

## The token question in detail

To understand why the announced shutdown attracts so much attention, you have to know where the token actually comes from and why it cannot simply be replaced.

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

### Could the community work around the shutdown?

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

This is where the three steps from the top of the article come from, each with its rationale:

- **Set it up now**, because token retrieval is the only step that necessarily runs through the Midea cloud. As long as the integration can still reach a working token interface, setup succeeds; after that, even a factory-fresh device will not help.
- **Back up the credentials**, because token and key are unlikely to be issued again after the shutdown. Home Assistant does store them locally, but a broken system, a failed restore, or an accidentally deleted integration would mean permanent loss of local control without an external backup.
- **Do not break the pairing without need**, because any factory reset and any removal from the Midea account can discard the keys stored on the device. The re-pairing that follows depends on exactly the cloud interfaces that are disappearing.

Day-to-day operation is not affected for now: local control uses the values already stored and no longer needs the token interfaces. A residual risk remains, however, if Midea, as the developer expects, eventually replaces the V1 LAN API through firmware updates as well. How to back up token, key, and configuration in practice is covered in the [second part](/en/blog/midea-portasplit-home-assistant-setup-and-hardening#backing-up-the-configuration).

## What this means for security

The shutdown is not only a question of availability; it has a security core. According to the `Midea AC LAN` warning, the older LAN architecture rests on a problematic assumption: the developers originally assumed that client communication was already sufficiently encrypted. As a result, the tokens issued by the cloud were given no expiry.

A non-expiring token is not a vulnerability in itself. It becomes a problem when it can be read out, appears in logs, ends up in backups, is passed to third parties, when the client encryption has been reconstructed, or when it cannot be revoked or rotated. The developer of `Midea AC LAN` writes that numerous Home Assistant plugins on GitHub use reconstructed client encryption methods. Combined with non-expiring tokens, that creates a security gap, which Midea is addressing by shutting down the token APIs and moving to a new architecture.

Precision of language matters here. The community integration does not "hack" the air conditioner. It implements a proprietary protocol that was reconstructed through reverse engineering. The security problem arises from long-lived secrets being usable and storable outside the app they were originally intended for.

For operation on your own network, what matters most is what token and key enable. Both authenticate the local communication with the device. If they fall into the wrong hands, an attacker could, depending on the protocol and their network position, discover the device, authenticate against it, read status information, change settings, switch the air conditioner on or off, change operating modes, and change the target temperature. To do so, the attacker still generally needs a network connection to the device; possessing token and key alone does not enable an attack from the open internet. Token and key are therefore to be treated like a password. How to embed the device on the network so that these values do little harm even after a slip-up is the subject of the [second part](/en/blog/midea-portasplit-home-assistant-setup-and-hardening#operating-the-portasplit-securely).

## Conclusion

Local control of the PortaSplit in Home Assistant stands and falls with a token and key that are currently obtainable only through the Midea cloud. This detour is not a cosmetic flaw but the consequence of a LAN protocol that ties local commands to cloud-derived credentials. Because Midea is shutting down exactly these interfaces and considers the underlying architecture a security problem, the long-term future of the unofficial local API is not guaranteed.

In practice: if you own or buy a PortaSplit, set it up soon, back up the credentials, and do not break the pairing lightly. Devices already set up keep running locally. The concrete setup, the choice of the right integration, and the network hardening are in the [second part: integrating and hardening the Midea PortaSplit in Home Assistant](/en/blog/midea-portasplit-home-assistant-setup-and-hardening).

## Sources

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> — The `Midea AC LAN` integration with the warning about the progressive shutdown of the cloud token APIs, the rationale around non-expiring tokens and reconstructed client encryption, and the description of the cloud-based token retrieval.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> — The `Midea Smart AC` integration: description of the cloud-based token and key retrieval on V3 devices and the local storage of the values.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome) — Vendor information on the SmartHome ecosystem and the referenced security and privacy standards.
