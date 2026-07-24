---
title: "Integrating the Midea PortaSplit into Home Assistant: Local Control, Setup, and Security Risks"
navTitle: "PortaSplit in HA"
description: "The Midea PortaSplit can be controlled locally from Home Assistant through a community integration. Which integration to pick, why the Midea cloud is still needed briefly during setup, what token and key really mean, and how to run the device securely on your home network."
date: "2026-07-24"
kategorie: "Smart Home & IoT"
timeToRead: "16 min to read"
themen:
  - "smart-home-iot"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
translationOf: "midea-portasplit-home-assistant"
slug: "midea-portasplit-home-assistant-integration"
url: "https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-integration"
aiPrompt: |
  You are my smart home assistant. Help me integrate a Midea PortaSplit into Home Assistant securely:
  1. First onboard the device normally with the MSmartHome app on a 2.4 GHz network.
  2. Install HACS and add the `Midea Smart AC` integration (mill1000/midea-ac-py).
  3. Add the device via discovery or manually (device ID, IP, port 6444, device type, token, key).
  4. Back up token, key, and the integration configuration encrypted and outside of Home Assistant.
  5. Set a DHCP reservation, move the device into a separate IoT VLAN, and write firewall rules so that only Home Assistant may reach it.
  6. Block outbound internet access as a test and verify across several restarts that local control stays stable.
  Warn me before any step that could destroy the existing pairing and force a new token retrieval through the Midea cloud.
---

![Example Home Assistant dashboard for a Midea PortaSplit showing room and target temperature, humidity, power draw, energy consumption, and compressor runtimes over the past 24 hours.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

The Midea PortaSplit is one of the more interesting portable air conditioners for rented flats: the compressor sits outside, the indoor unit runs comparatively quietly, and installation requires no structural work. Less well known is that the PortaSplit can also be integrated into Home Assistant, which allows it to be controlled based on room temperature, electricity price, window state, or presence.

After setup, the integration works largely locally on your own network. It does not, however, come entirely without a cloud and entirely without security questions. One current warning from the Midea AC LAN project is particularly relevant: Midea is progressively shutting down the cloud interfaces that Home Assistant uses to retrieve the credentials needed for local communication. At the same time, the developer describes the existing token architecture as a security problem.

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

Once set up, both communicate with the air conditioner directly over the local network. On newer Midea devices, however, the cloud is needed once to obtain a device-specific token and key.

For the PortaSplit, `Midea Smart AC` is currently the more obvious choice. That integration lists the PortaSplit explicitly as a supported device and now also handles the "Out Silent Mode" introduced in 2026. `Midea AC LAN` is the alternative, covering numerous other Midea device types besides air conditioners.

## What the integration gives you

After setup, the PortaSplit appears as a `climate` entity in Home Assistant. Depending on firmware and integration, the following functions are available:

- power on and off
- set the target temperature
- read the current room temperature
- cooling, dehumidifying, and fan-only operation
- set fan speed
- control the swing function
- eco and boost mode
- read humidity
- show error codes
- read energy and power values
- show compressor values
- enable the outdoor unit's silent mode

Which entities actually appear depends on the model, the firmware, the protocol in use, and the integration. `Midea Smart AC` queries the capabilities reported by the device and hides functions the model does not support. `Midea AC LAN` also documents extensive climate entities, including temperature, humidity, current power, total energy, compressor frequency, pump status, and various operating modes, and names dedicated decoding methods for the energy data of certain PortaSplit subtypes.

Not every reading shown is necessarily correct. Energy consumption and power in particular are transmitted in different formats across Midea models. If Home Assistant shows obviously wrong values, the decoding method usually needs adjusting rather than the device being faulty.

## Why local control still needs a cloud

This is the key technical difference between "local control" and "a setup entirely free of the cloud". After setup, the actual control commands go directly from Home Assistant to the PortaSplit:

```text
Home Assistant → local network → Midea PortaSplit
```

That means a switching command does not have to travel through an external Midea server, response times are short, an outage of the Midea cloud does not necessarily break an already configured local setup, and the device generally remains controllable without internet access.

On newer devices using the so-called V3 protocol, however, Home Assistant needs two device-specific values: a token and a key. Both authenticate and encrypt the local communication. The integration does not know them automatically, so it retrieves them through a Midea cloud interface during initial setup. The developers of `Midea Smart AC` state this explicitly: for V3 devices, the Midea cloud is used during discovery to obtain token and key. Those are then stored locally, and no cloud connection is required for further control.

Simplified, the sequence looks like this:

```text
1. PortaSplit is paired with MSmartHome
2. Home Assistant signs in to a Midea cloud
3. Home Assistant receives device ID, token, and key
4. Token and key are stored locally
5. Home Assistant controls the PortaSplit directly over the LAN
```

"Locally controllable" therefore does not automatically mean "never connected to a cloud".

## Which integration fits

### Midea Smart AC

The <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> repository focuses on Midea air conditioners and related OEM models and supports device types `0xAC` and `0xCC`. It offers local control, a graphical setup flow, automatic discovery, manual setup with token and key, and automatic capability detection. The PortaSplit's "Out Silent Mode" is explicitly supported.

As an indicator of compatibility, the project names apps including Artic King, Midea Air, NetHome Plus, SmartHome or MSmartHome, Toshiba AC NA, and 美的美居. In Europe the PortaSplit typically uses MSmartHome and therefore fits into this ecosystem.

### Midea AC LAN

The <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> repository supports not only air conditioners but numerous other Midea device classes: dehumidifiers, fans, air purifiers, washing machines, dryers, dishwashers, water heaters, heat pumps, refrigerators, and more, partly under third-party brands such as Carrier or Electrolux. It also offers local communication, automatic device discovery, and additional sensors, and according to the project description keeps a long-lived TCP connection to the device open in order to synchronise state changes promptly. It requires Home Assistant 2024.4.1 or newer.

Its biggest drawback right now is the developer's warning: the cloud token APIs used for adding new devices are being shut down step by step. Adding new devices may become impossible as a result.

### Recommendation

For a PortaSplit-only installation I would start with `Midea Smart AC` and keep `Midea AC LAN` in mind as the alternative. `Midea Smart AC` is more narrowly tailored to air conditioners and documents the current PortaSplit features explicitly.

Running both integrations against the same device permanently makes no sense. Multiple parallel connections lead to state problems, unnecessary network traffic, and behaviour that is hard to trace.

## Prerequisites

You need a Midea PortaSplit with Wi-Fi, a 2.4 GHz network, the MSmartHome app, a Midea account, Home Assistant, HACS, and network access between Home Assistant and the PortaSplit. Pair the PortaSplit with the MSmartHome app first, and only then add it to Home Assistant.

## Step 1: Pair the PortaSplit with MSmartHome

1. Install the MSmartHome app.
2. Create a Midea account or sign in.
3. Put the PortaSplit into Wi-Fi pairing mode.
4. Connect the device to the 2.4 GHz network.
5. Verify that the PortaSplit can be controlled from the app.

Many IoT devices still support 2.4 GHz only. If the router uses the same SSID for 2.4 and 5 GHz, setup usually still works. If it does not, providing a separate 2.4 GHz network temporarily helps.

## Step 2: Install HACS

HACS is the Home Assistant Community Store. It installs community integrations that are not part of Home Assistant Core. After installing HACS, open it, switch to the integrations section, search for `Midea Smart AC`, download the integration, and restart Home Assistant. Alternatively, search for `Midea AC LAN`.

HACS simplifies installation and updates. It does not, however, turn a custom integration into an officially reviewed Home Assistant component. That distinction matters from a security perspective and is covered below.

## Step 3: Add Midea Smart AC

After the restart, go to Settings, Devices & Services, and Add Integration, search for `Midea Smart AC`, and choose `Discover devices`. The integration can either scan the entire local network or target the PortaSplit's IP address directly.

Once the device is found, newer V3 devices require region, Midea account, password, and device ID, along with the token and key derived from them. The cloud region has to match the account in use. If that fails, the project recommends trying the other regions on offer as well.

### Manual setup

If automatic setup fails, the device can be configured manually. `Midea Smart AC` needs the following details:

```text
Device ID
IP address
Port
Device type
Token
Key
```

The documented default port is:

```text
6444/TCP
```

For V3 devices the documentation specifies the token as a 128-character and the key as a 64-character hexadecimal string. Both values are secrets and have to be treated as such. If you would rather not obtain the credentials through discovery, you can retrieve them with your own account using the `msmart-ng` CLI.

## The Midea AC LAN warning

The `Midea AC LAN` repository has carried a prominently placed warning since 2026. According to the developer, Midea has already closed the server-side token APIs in the Meiju cloud and the SmartHome cloud. The integration therefore currently falls back to the token interfaces of the NetHome Plus cloud, and those are expected to be closed progressively too. The consequence would be that already configured devices keep working locally, while new devices can no longer be added. The developer goes further, writing that Midea intends to move to a new cloud control API in the long run and thereby render the existing V1 LAN API unusable.

This needs a differentiated reading. It is the assessment of an open-source project, not a binding roadmap from Midea, and the timeline is unknown. A future firmware update may change local functionality, and a stored token may keep working, but not necessarily forever. A factory reset, a Wi-Fi module swap, or a new device may require obtaining a token again.

Anyone who wants to use the integration should therefore not only set the PortaSplit up but also back up the local credentials and the Home Assistant configuration.

## Security analysis

### The token problem

According to the `Midea AC LAN` warning, the older LAN architecture rests on a problematic assumption: the developers originally assumed that client communication was already sufficiently encrypted. As a result, the tokens issued by the cloud were given no expiry.

```text
Token issued once → potentially valid indefinitely
```

A non-expiring token is not a vulnerability in itself. It becomes a problem when it can be read out, appears in logs, ends up in backups, is passed to third parties, when the client encryption has been reconstructed, or when it cannot be revoked or rotated. The developer of `Midea AC LAN` writes that numerous Home Assistant plugins on GitHub use reconstructed client encryption methods. Combined with non-expiring tokens, that creates a security gap, which Midea is addressing by shutting down the token APIs and moving to a new architecture.

Precision of language matters here. The community integration does not "hack" the air conditioner. It implements a proprietary protocol that was reconstructed through reverse engineering. The security problem arises from long-lived secrets being usable and storable outside the app they were originally intended for.

### What token and key make possible

Token and key authenticate local communication with the device. If both values fall into the wrong hands, an attacker could, depending on the protocol and their network position, discover the device, authenticate against it, read status information, change settings, switch the air conditioner on or off, change operating modes, and change the target temperature.

To do so, the attacker still generally needs a network connection to the device. Possessing token and key alone does not enable an attack from the open internet. The risk does rise considerably, though, if the PortaSplit has been made reachable from the internet, port forwards exist, Home Assistant is compromised, an attacker has access to the local Wi-Fi, the IoT network is insufficiently segmented, backups sit unencrypted in public, or secrets are uploaded to a GitHub repository. Token and key are therefore to be treated like a password.

### The JSON file is not a harmless backup

After a successful setup of a V3 device, `Midea AC LAN` writes a JSON configuration file. The documented path is:

```text
/config/.storage/midea_ac_lan/
```

The file is named after the device ID:

```text
<device-id>.json
```

The project explicitly recommends backing this file up outside the Home Assistant system so that a fresh setup remains possible later, should the cloud token APIs no longer be available. The file is not an ordinary text note, though. It can contain device ID, serial number, IP address, token, key, protocol information, and cloud or device parameters. Accordingly:

```text
Do not upload it to a public GitHub repository.
Do not post it in forums.
Do not share it as an unredacted screenshot.
Do not send it by unencrypted email.
```

A private Git repository is not automatically the right place either, because secrets remain in the Git history even after they have been deleted from the current file. Better options are an encrypted backup, a password manager with file attachments, an encrypted NAS backup, encrypted offline media, or an encrypted archive with the password stored separately.

### Debug logs contain sensitive data

When problems arise, open-source projects frequently ask for debug logs. The `Midea AC LAN` documentation shows how to enable logging for the two relevant components:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

The logs can then be downloaded via Settings, System, and Logs. Depending on the integration and the failure case, such logs can contain local IP addresses, device ID, serial number, model identifier, cloud responses, account information, tokens or parts of them, network packets, and timestamps revealing usage patterns. Before uploading them to a public GitHub issue, review them and redact sensitive values.

Once troubleshooting is finished, remove the debug logging again. Permanently enabled debug logging not only increases storage consumption, it also increases the amount of sensitive information in backups.

### HACS and the supply chain risk

`Midea Smart AC` and `Midea AC LAN` are custom integrations. They run inside Home Assistant and therefore have far-reaching access to its runtime environment. A malicious or compromised integration could in theory read configuration data, read out secrets, open network connections, scan devices on the local network, read the state of other entities, transmit data to external systems, and affect the availability of Home Assistant.

That does not mean the integrations named here are malicious. Both projects are publicly auditable, actively developed, and have a visible community. Open source is not an automatic security guarantee, though. Before installing, it is worth checking at least whether the repository is actively maintained, whether releases appear regularly, how many people contribute to the code, whether there are open security issues, whether maintainers or repository owners changed recently, whether HACS points to the expected repository, and whether an update contains unusually large or unexplained changes.

Updates should not be installed blindly the moment they are published. For security-critical smart home systems in particular, waiting a few days and reviewing release notes and reported problems is sensible.

### Home Assistant as the central trust anchor

Controlling the PortaSplit locally shifts part of the trust from the Midea cloud to Home Assistant. If Home Assistant is compromised, an attacker may control not just the air conditioner but the entire smart home.

Home Assistant should therefore be updated regularly, not published through an unprotected port forward, protected with a strong and unique password, secured with multi-factor authentication, backed up in encrypted form, kept free of unnecessary add-ons, and given no unnecessary SSH access from the internet. For remote access, a VPN, Home Assistant Cloud, or a properly configured reverse proxy are better options than a simple port forward to port 8123.

### No port forwarding to the PortaSplit

The most common avoidable mistake would be exposing the local device port to the internet. A rule like this would be dangerous:

```text
Internet → TCP 6444 → PortaSplit
```

There is no good reason to make the PortaSplit reachable from the internet directly. Home Assistant already sits on the local network and acts as the controlling instance. The router should have no port forward to the PortaSplit, restrict or disable UPnP where possible, block inbound connections by default, and use no DMZ exposure for the device.

### A dedicated IoT VLAN

The best network architecture is a separate IoT network:

```text
VLAN 10: trusted clients
VLAN 20: servers and Home Assistant
VLAN 30: IoT devices
VLAN 40: guests
```

The PortaSplit lives in the IoT VLAN. Home Assistant is allowed to reach the device specifically, while the PortaSplit must not reach PCs, NAS, and other internal systems freely. One possible firewall logic:

```text
Home Assistant → PortaSplit: allow
PortaSplit → Home Assistant: allow established connections
PortaSplit → internal clients: block
PortaSplit → NAS: block
PortaSplit → management network: block
Internet → PortaSplit: block
```

During initial setup the device needs internet access to the Midea cloud. Once local setup has succeeded, you can test whether outbound internet access can be blocked. Do not set a final block right away. Check first whether local control still works, whether the device stays reachable after a restart, whether it survives a router reboot, whether it still responds after several days, whether the MSmartHome app is still needed, and whether firmware updates are still offered. If you want to keep using the cloud and firmware updates, allow outbound internet access temporarily and block it again afterwards.

### Network segmentation can break discovery

Automatic device discovery frequently relies on broadcast or multicast traffic, and that is normally not routed across VLAN boundaries. Home Assistant may therefore fail to find the PortaSplit automatically even though a regular IP connection would be permitted.

In that case it helps to set the PortaSplit up temporarily in the same VLAN as Home Assistant, to enter the device IP manually, to use a suitable broadcast relay function, or to define targeted firewall rules after setup. Manual configuration is often the better option from a security perspective, because it requires no additional broadcast traffic between networks.

### Static DHCP assignment

The PortaSplit should get a fixed DHCP reservation on the router:

```text
PortaSplit → 192.168.30.25
```

A DHCP reservation is usually preferable to a static IP configured on the device itself. Home Assistant finds the device reliably, firewall rules can be scoped to a fixed address, troubleshooting becomes easier, and the assignment survives router or device restarts. A firewall rule can then be written very narrowly:

```text
Home Assistant IP → 192.168.30.25:6444/TCP
```

Verify the port actually required against your integration and your device.

### Securing the cloud account

As long as the Midea cloud is used for setup or app control, the Midea account remains part of the security model. That means a unique password not shared with other services, a password manager, multi-factor authentication where offered, removing old phones and sessions, avoiding shared accounts, and regularly reviewing which devices are registered to the account.

If the Home Assistant integration asks for username and password during setup, check whether the credentials are stored only for the one-off token retrieval or permanently. The developers of `Midea Smart AC` write that devices are not linked to built-in integration accounts after setup, and that token and key can also be obtained manually with your own account via the CLI. Where possible, your own account is preferable to shared or built-in collective accounts.

### Block the cloud or not?

Once setup is complete, the question arises whether the PortaSplit's internet access should be blocked entirely. Arguments in favour are less telemetry, less dependence on external services, a smaller attack path through the vendor cloud, the fact that the device cannot contact arbitrary external destinations, and a reduced impact of cloud-side changes.

Arguments against are that the MSmartHome app may stop working outside the home network, that firmware updates are no longer downloaded, that time or cloud functions may fail, that re-registration or recovery becomes harder, and that some devices behave unexpectedly after a long time offline.

A pragmatic order of operations: set the device up normally, test Home Assistant and the app, back up token and configuration, block internet access, restart the device and Home Assistant, observe for several days, and re-enable internet access temporarily if needed.

### Firmware updates: security gain or integration risk?

Firmware updates are a dilemma on IoT devices. They can close known vulnerabilities, improve stability, modernise security mechanisms, and add features. They can also change local interfaces, break reverse-engineered integrations, invalidate tokens, disable the local API, and introduce new cloud dependencies.

The PortaSplit firmware shipped in January 2026, for example, brought a new silent mode for the outdoor unit that reduces noise by roughly 6 decibels. Community integrations had to reconstruct and implement it first, documented in a dedicated GitHub issue for the PortaSplit.

The conclusion: do not block firmware updates on principle, check before an update whether other Home Assistant users are reporting problems, back up configuration and token beforehand, create a Home Assistant backup, and test local control thoroughly after the update. Security does not mean "never update". Outdated firmware can be more dangerous than a temporarily incompatible integration.

### What Midea says about security

Midea markets its SmartHome ecosystem with reference to several security and privacy standards, naming EN 303 645, UK PSTI, NIST, GDPR-compliant data processing, and the requirements of the EU Radio Equipment Directive. Those are positive signals, but they say nothing about how each individual PortaSplit firmware, each cloud endpoint, and each local API is actually implemented. Certification and marketing statements do not replace a technical review of the specific device.

It would be equally wrong to conclude from a community integration's warning that the PortaSplit is generally insecure. The problem described concerns the architecture of long-lived tokens and their use by unofficial clients.

### Risk by scenario

| Scenario | Risk | Rationale |
| --- | --- | --- |
| Ordinary home network without port forwarding | manageable | An attacker first needs access to the Wi-Fi, Home Assistant, or a backup. |
| Flat home network with many insecure IoT devices | medium | Another compromised IoT device can reach the PortaSplit or Home Assistant on the same network. |
| PortaSplit reachable from the internet | high | The device should never be published through a port forward. |
| Token and key public on GitHub | high | The secrets have to be treated as compromised; whether they can be revoked is not guaranteed. |
| Separate IoT VLAN, restrictive firewall, local control | comparatively low | Even with a vulnerability in the device, lateral movement is heavily constrained. |

## Backing up the configuration

When using `Midea AC LAN`, back up the JSON file after a successful setup. From the Home Assistant terminal:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Show the file:

```bash
cat <device-id>.json
```

Do not transfer the file through a public web service when copying it. An encrypted archive, later moved into an encrypted backup, is the better option:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

The files in `.storage` should not be edited manually. The developer explicitly recommends neither deleting nor directly modifying the JSON file when problems occur, but renaming and backing it up before making changes.

A full Home Assistant backup contains these files as well. A separate copy is still worthwhile, because Home Assistant backups can become corrupted, a restore can overwrite the integration, the file may be needed specifically for a later fresh setup, and a backup should never live only on the same system.

## Removing secrets from a published Git repository

If a JSON file was published on GitHub by accident, deleting it and pushing a new commit is not enough. The file remains retrievable from the Git history. At minimum, these steps are required:

1. Make the repository private immediately, if possible.
2. Remove the file from the entire Git history.
3. Account for GitHub caches and forks.
4. Treat the token as compromised.
5. Remove the device from the Midea account and pair it again, if that generates new keys.
6. Set the Home Assistant integration up again.
7. Change the Midea account password if credentials were affected too.

Whether re-pairing actually generates a new token varies by device and cloud architecture. Do not rely on a change of account password automatically invalidating the local device token.

## Useful automations

Once integrated, the PortaSplit can be run far more intelligently. Adapt the entity IDs to your own installation.

Cool only when the windows are closed:

```yaml
alias: PortaSplit only with windows closed
triggers:
  - trigger: state
    entity_id: binary_sensor.living_room_window
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Switch on at high room temperature:

```yaml
alias: PortaSplit on when hot
triggers:
  - trigger: numeric_state
    entity_id: sensor.living_room_temperature
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.living_room_window
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Pre-cool before bedtime:

```yaml
alias: Pre-cool the bedroom
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.bedroom_temperature
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Switch off when nobody is home:

```yaml
alias: PortaSplit off when away
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Recommended configuration at a glance

```text
1. Set the PortaSplit up with MSmartHome
2. Install Midea Smart AC through HACS
3. Add the PortaSplit via discovery or manually
4. Create a DHCP reservation
5. Take a Home Assistant backup
6. Back up token and configuration data encrypted
7. Move the PortaSplit into a separate IoT VLAN
8. Allow access from Home Assistant to the PortaSplit
9. Block access from the PortaSplit to internal networks
10. Block internet access as a test
11. Verify local control after restarts
12. Apply firmware and integration updates in a controlled way
```

The intended direction of communication then looks like this:

```text
Home Assistant
    │
    │ specifically allowed
    ▼
Midea PortaSplit
    │
    ├── no access to PCs
    ├── no access to NAS
    ├── no access to the management network
    └── internet only when needed
```

## Conclusion

The Midea PortaSplit integrates into Home Assistant surprisingly well. Once configured, it is locally controllable and can be wired into automations, which removes a large part of the cloud dependency from day-to-day operation.

The path is not entirely cloud-free, though: newer devices need a token and key during initial setup, currently obtained through Midea cloud interfaces. Those are exactly the interfaces the `Midea AC LAN` project says are being shut down step by step. The long-term future of the unofficial local API is therefore not guaranteed. If you already own a PortaSplit, set it up soon and back up the configuration. If you are buying one, be aware that Home Assistant support rests on community work and a proprietary protocol reconstructed through reverse engineering.

From a security perspective, the integration is defensible as long as a few ground rules hold: no port forwarding, keep token and key secret, encrypt backups, review debug logs before publishing them, secure Home Assistant, segment IoT devices, restrict outbound internet access to what is necessary, and do not install firmware and HACS updates blindly. Run that way, the PortaSplit is not only a capable air conditioner but a sensible component of a locally controlled smart home.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> — The `Midea Smart AC` integration: supported device types `0xAC` and `0xCC`, PortaSplit with "Out Silent Mode", cloud usage for token and key retrieval on V3 devices, manual configuration, and default port 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> — The `Midea AC LAN` integration with the warning about the progressive shutdown of the cloud token APIs, the rationale around non-expiring tokens and reconstructed client encryption, and the Home Assistant 2024.4.1 minimum version.

3.  [midea_ac_lan: climate entity documentation](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md) — Entities and attributes for air conditioners, including power, total energy, compressor frequency, and the energy decoding methods for individual subtypes.

4.  [midea_ac_lan: debug and configuration notes](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md) — Device configuration stored under `/config/.storage/midea_ac_lan/`, the recommendation to back up rather than delete the JSON file, and the logger configuration for debug logs.

5.  [Issue 779: PortaSplit Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779) — Request for support of the outdoor unit silent mode introduced with the January 2026 firmware update, which reduces noise by roughly 6 decibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome) — Vendor statements on the security and privacy standards EN 303 645, PSTI, NIST, GDPR, and RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/) — Installing and managing custom integrations that are not part of Home Assistant Core.
