---
title: "Control Midea PortaSplit locally with Home Assistant and operate it securely"
navTitle: "Set up PortaSplit"
description: "From the right community integration to an IoT VLAN: how to set up PortaSplit, protect tokens and keys, and restrict cloud and network access."
date: "2026-07-24"
kategorie: "Home Assistant and IoT"
timeToRead: "14 min read"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - serverloser-newsletter-cloudflare-workers-d1
slug: "controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura"
translationOf: "midea-portasplit-home-assistant-einrichten"
translationId: article-36e7710abe426781
translationReview: required
translationSourceHash: 859c24ec38af3b4b931702c7be50cf2224580d30045559ba089224d0de25339c
translatedAt: 2026-08-08T14:20:18.797Z
url: https://rafaelpfister.ch/es/blog/controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura
translationModel: gpt-5.6-terra
---

The Midea PortaSplit can be controlled directly on the local network via Home Assistant after setup. To do so, the community integration requires two device-specific credentials from the Midea Cloud: a token and a key.

This article guides you through selecting, setting up and securing the integration. The solutions described come from the community and are not officially supported by either Midea or Home Assistant. Firmware or cloud changes may therefore affect their behaviour at any time. Background information on the token interface and the ambiguous shutdown warning can be found in the [analysis of the Midea Cloud APIs](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## How local control works

Once configured, the actual control commands are sent directly from Home Assistant to the PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

A switching command does not need to go through an external Midea server, response times are short, an outage of the Midea Cloud does not necessarily disable already configured local control, and the device can generally still be controlled without internet access.

However, on newer devices using the so-called V3 protocol, the PortaSplit does not accept local commands without protection. Home Assistant requires two device-specific values, a token and a key, which are used to authenticate and encrypt the local connection. During the initial setup, the integration retrieves them once through a Midea Cloud interface and then stores them locally; no cloud connection is required for subsequent control.

In simplified form, the process looks like this:

1. PortaSplit is connected to MSmartHome.
2. Home Assistant logs in to a Midea Cloud.
3. Home Assistant receives the device ID, token and key.
4. Token and key are stored locally.
5. Home Assistant controls the PortaSplit directly on the LAN.

## Which integration is suitable

### Midea Smart AC

The repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> focuses on Midea air conditioners and related OEM models and supports the device types `0xAC` and `0xCC`. It offers local control, graphical setup, automatic discovery, manual setup with token and key, and automatic querying of device capabilities. The PortaSplit's “Out Silent Mode” is explicitly supported.

As an indication of compatibility, the project names the apps Artic King, Midea Air, NetHome Plus, SmartHome or MSmartHome, Toshiba AC NA and 美的美居, among others. In Europe, the PortaSplit typically uses MSmartHome and therefore fits into this ecosystem.

### Midea AC LAN

The repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> supports not only air conditioners, but also numerous other Midea device classes: dehumidifiers, fans, air purifiers, washing machines, dryers, dishwashers, water heaters, heat pumps, refrigerators and more, in some cases under third-party brands such as Carrier or Electrolux. It also offers local communication, automatic device discovery and additional sensors and, according to the project description, keeps a longer TCP connection to the device open to synchronise status changes promptly. Home Assistant 2024.4.1 or later is required.

The biggest disadvantage at present is the developer's warning: the cloud token APIs used to add new devices are being phased out. This may make it impossible to add new devices later.

### Recommendation

For a PortaSplit-only installation, I would start with `Midea Smart AC` and keep `Midea AC LAN` in mind as an alternative. `Midea Smart AC` is more tightly focused on air conditioners and explicitly documents the current PortaSplit functions.

It is not advisable to operate both integrations simultaneously and permanently with the same device. Multiple parallel connections lead to status issues, unnecessary network traffic and behaviour that is difficult to trace.

## What the integration provides

After setup, the PortaSplit appears as a `climate` entity in Home Assistant. Depending on the firmware and integration, the following functions are available, among others:

- Switch on and off
- Set target temperature
- Read the current room temperature
- Cooling, dehumidification and fan-only operation
- Set fan speed
- Control the swing function
- Eco and boost mode
- Read humidity
- Display error codes
- Read energy and power values
- Display compressor values
- Activate the outdoor unit's quiet mode

Which entities actually appear depends on the model, firmware, protocol used and the relevant integration. `Midea Smart AC` queries the capabilities reported by the device and hides functions that the model does not support. `Midea AC LAN` also documents extensive climate entities, including temperature, humidity, current power, total energy, compressor frequency, pump status and various operating modes, and specifies its own methods for decoding energy data for certain PortaSplit subtypes.

Not every displayed measurement is necessarily correct. Energy consumption and power in particular are transmitted in different formats by different Midea models. If Home Assistant shows obviously incorrect values, the decoding method in use usually needs to be adjusted rather than the device being defective.

## Requirements

You need a Midea PortaSplit with Wi-Fi functionality, a 2.4 GHz Wi-Fi network, the MSmartHome app, a Midea user account, Home Assistant, HACS and network access between Home Assistant and PortaSplit. The PortaSplit should first be connected normally using the MSmartHome app, then added to Home Assistant.

## Step 1: Connect PortaSplit to MSmartHome

1. Install the MSmartHome app.
2. Create a Midea account or sign in.
3. Put the PortaSplit into Wi-Fi pairing mode.
4. Connect the device to the 2.4 GHz Wi-Fi network.
5. Check whether the PortaSplit can be controlled through the app.

Many IoT devices still support only 2.4 GHz. If the router uses the same SSID for 2.4 and 5 GHz, setup will usually still work. If you encounter problems, it helps to provide a separate 2.4 GHz Wi-Fi network temporarily.

## Step 2: Install HACS

HACS is the Home Assistant Community Store. It lets you install community integrations that are not part of Home Assistant Core. After installing HACS, open HACS, go to Integrations, search for `Midea Smart AC`, download the integration and restart Home Assistant. Alternatively, you can search for `Midea AC LAN`.

HACS simplifies installation and updates. However, it does not make a custom integration an officially reviewed Home Assistant component. This distinction is significant from a security perspective and is discussed below.

## Step 3: Add Midea Smart AC

After restarting, go to Settings, Devices & Services and Add Integration, then search for `Midea Smart AC` and select `Discover devices`. The integration can either scan the entire local network or query the PortaSplit's IP address directly.

If the device is found, the integration requires the region, Midea account, password and device ID for newer V3 devices, as well as the token and key derived from them. The cloud region must match the account in use. In case of problems, the project recommends trying the other available regions as well.

### Manual setup

If automatic setup fails, the device can be configured manually. `Midea Smart AC` requires the following details:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

The documented default port is:

```text
6444/TCP
```

For V3 devices, the documentation specifies the token as a 128-character hexadecimal string and the key as a 64-character hexadecimal string. Both values are secrets and must be treated accordingly. If you do not want to obtain the credentials through discovery, you can retrieve them with your own account using the `msmart-ng` CLI.

## Operating the PortaSplit securely

Anyone controlling the PortaSplit locally regains part of the control from the manufacturer's cloud, but also shifts responsibility to their own network. The following measures ensure that a token or key causes little damage even in the event of an incident and that the device remains properly isolated.

### Token and key are secrets

The token and key authenticate local communication with the device and must be treated like a password. Above all, this means they do not belong in logs, unencrypted backups or a repository.

### No port forwarding to the PortaSplit

The most common avoidable mistake would be making the local device port directly accessible from the internet. A rule like this would be dangerous:

```text
Internet → TCP 6444 → PortaSplit
```

There is no good reason to make the PortaSplit directly accessible from the internet. Home Assistant is already on the local network and acts as the controlling instance. The router should have no port forwarding to the PortaSplit, should restrict or disable UPnP where possible, block incoming connections by default and not use a DMZ rule for the device.

### Dedicated IoT VLAN

The best network architecture is a separate IoT network:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

The PortaSplit is located in the IoT VLAN. Home Assistant may access the device in a targeted manner, but the PortaSplit must not be able to access PCs, NAS devices and other internal systems freely. One possible firewall policy is:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

During initial setup, the device requires internet access to the Midea Cloud. After local setup is successful, you can test whether outbound internet access can be blocked. Do not apply a final block immediately. First check whether local control still works, whether the device remains reachable after a restart, whether it survives a router restart, whether it still responds after several days, whether the MSmartHome app is still needed and whether firmware updates are still offered. If you want to continue using cloud features and firmware updates, you can temporarily allow outbound internet access and block it again afterwards.

### Network segmentation can prevent discovery

Automatic device discovery often relies on broadcast or multicast traffic, which is normally not routed across VLAN boundaries. Home Assistant may therefore fail to discover the PortaSplit automatically even if a regular IP connection would be allowed.

In that case, it helps to set up the PortaSplit temporarily in the same VLAN as Home Assistant, specify the device IP manually, use a suitable broadcast relay function or define targeted firewall rules after setup. Manual configuration is often even the better option from a security perspective because no additional broadcast traffic needs to be allowed between the networks.

### Static DHCP assignment

The PortaSplit should receive a fixed DHCP assignment on the router:

```text
PortaSplit → 192.168.30.25
```

A DHCP reservation is generally preferable to a static IP configured on the device. Home Assistant reliably finds the device, firewall rules can be restricted to a fixed address, troubleshooting becomes easier and the assignment remains stable after router or device restarts. This allows a firewall rule to be formulated very narrowly:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

The port actually required must be verified based on the integration and your own device.

### Home Assistant as the central trust anchor

Anyone controlling the PortaSplit locally partly shifts trust from the Midea Cloud to Home Assistant. If Home Assistant is compromised, an attacker may control not only the air conditioner but the entire smart home.

Home Assistant should therefore be updated regularly, not exposed through unprotected port forwarding, protected with a strong, unique password, use multi-factor authentication, create encrypted backups, contain only necessary add-ons and not allow unnecessary SSH access from the internet. For remote access, a VPN, Home Assistant Cloud or a properly configured reverse proxy are better options than simple port forwarding to port 8123.

### HACS and supply-chain risk

`Midea Smart AC` and `Midea AC LAN` are custom integrations. They run inside Home Assistant and therefore receive extensive access to its runtime environment. A malicious or compromised integration could theoretically read configuration data, extract secrets, establish network connections, scan devices on the local network, read states of other entities, transmit data to external systems and impair Home Assistant availability.

This does not mean that the integrations mentioned are malicious. Both projects are publicly visible, actively developed and have a visible community. However, open source is not an automatic security guarantee. Before installation, it is worth checking at least whether the repository is actively maintained, whether there are regular releases, how many people contribute code, whether there are open security issues, whether maintainers or repository owners have recently changed, whether HACS points to the expected repository and whether an update contains unusually large or unexplained changes.

Updates should not be installed blindly immediately after release. Especially for security-critical smart-home systems, it makes sense to wait a few days and review release notes and reported issues.

### Secure the cloud account

As long as the Midea Cloud is used for setup or app control, the Midea account remains part of the security model. It should have a unique password that is not shared with other services, a password manager, multi-factor authentication where offered, old smartphones and sessions removed, no shared accounts and regular checks of which devices are registered in the account.

If the Home Assistant integration requests a username and password during setup, check whether the credentials are used only for the one-time token retrieval or stored permanently. The developers of `Midea Smart AC` state that devices are not linked to built-in integration accounts after setup and that the token and key can also be obtained manually through the CLI using your own account. Where possible, your own account should be preferred over third-party or integrated shared accounts.

### Block the cloud or not?

After successful setup, the question arises whether the PortaSplit's internet access should be blocked completely. Arguments in favour of blocking include less telemetry, reduced dependence on external services, a smaller attack path through the manufacturer's cloud, the fact that the device cannot contact arbitrary external destinations and reduced impact from cloud-side changes.

Arguments against it are that the MSmartHome app may no longer work outside the home network, firmware updates may no longer download, time or cloud functions may fail, signing in again or restoring access may become more difficult, and some devices may react unexpectedly after being offline for a long time.

A pragmatic sequence is: set up the device normally, test Home Assistant and the app, back up the token and configuration, block internet access, restart the device and Home Assistant, observe them for several days and, if necessary, temporarily re-enable internet access only.

### Firmware updates: security gain or integration risk?

Firmware updates are a dilemma for IoT devices. They can close known vulnerabilities, improve stability, modernise security mechanisms and add new functions. But they can also change local interfaces, break reverse-engineered integrations, invalidate tokens, disable the local API and introduce new cloud dependencies.

For example, the PortaSplit firmware released in January 2026 introduced a new quiet mode for the outdoor unit that reduces noise by around 6 decibels. The community integrations first had to understand and implement it, as documented in a dedicated GitHub issue for the PortaSplit.

This means that firmware updates should not be prevented categorically: before an update, check whether other Home Assistant users report problems, back up the configuration and token beforehand, create a Home Assistant backup and fully test local control after the update. Security does not mean “never update”. Outdated firmware can be more dangerous than a temporarily incompatible integration.

### Debug logs contain sensitive data

When problems occur, open-source projects often request debug logs. The documentation for `Midea AC LAN` shows how to enable logging for the two relevant components:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

You can then download the logs through Settings, System and Logs. Depending on the integration and the error, such logs may contain local IP addresses, device ID, serial number, model identifier, cloud responses, account information, token or parts of it, network packets, as well as timestamps and usage behaviour. Before uploading them to a public GitHub issue, review and redact sensitive values.

Once troubleshooting is complete, debug logging should be removed again. Permanently enabled debug logging not only increases storage consumption, it also increases the amount of sensitive information in backups.

### What Midea itself says about security

Midea promotes its SmartHome ecosystem as being aligned with several security and data-protection standards, including EN 303 645, UK PSTI, NIST, GDPR-compliant data processing and the requirements of the EU Radio Equipment Directive. These are positive signals, but they do not say how every individual PortaSplit firmware, cloud endpoint and local API is actually implemented. Certification and marketing claims do not replace a technical assessment of the specific device.

Likewise, it would be wrong to infer from a community integration warning that the PortaSplit is generally insecure. The described issue concerns the architecture of long-lived tokens and their use by unofficial clients.

### Risk by scenario

| Scenario | Risk | Reason |
| --- | --- | --- |
| Normal home network without port forwarding | manageable | An attacker must first gain access to Wi-Fi, Home Assistant or a backup. |
| Flat home network with many insecure IoT devices | medium | Another compromised IoT device can reach PortaSplit or Home Assistant on the same network. |
| PortaSplit directly accessible from the internet | high | The device should never be exposed through port forwarding. |
| Token and key public on GitHub | high | The secrets must be considered compromised; whether they can be revoked is not guaranteed. |
| Separate IoT VLAN, restrictive firewall, local control | comparatively low | Even if the device has a vulnerability, its freedom of movement on the network is severely restricted. |

## Back up the configuration

Backing up the token, key and configuration is the most important one-time step: once the cloud token interfaces are closed, a backup is the only way to set up the device again. `Midea AC LAN` stores a JSON configuration file for V3 devices after successful setup. The documented path is:

```text
/config/.storage/midea_ac_lan/
```

The file uses the device ID as its filename:

```text
<device-id>.json
```

This file is not an ordinary text note. It may contain the device ID, serial number, IP address, token, key, protocol information and cloud and device parameters. Accordingly:

- Do not upload it to a public GitHub repository.
- Do not post it in forums.
- Do not share it as an unredacted screenshot.
- Do not send it by unencrypted email.

Even a private Git repository is not automatically the right storage location because secrets remain in Git history even if they are later removed from the current file. Better options include an encrypted backup, a password manager with file attachment, an encrypted NAS backup, encrypted offline media or an encrypted archive with the password stored separately.

To back it up through the Home Assistant terminal:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Display the file:

```bash
cat <device-id>.json
```

To copy it, do not transfer the file through a public web service. An encrypted archive, subsequently moved to an encrypted backup, is preferable:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

The files in `.storage` should not be edited manually. The developer explicitly recommends neither deleting nor directly modifying the JSON file in case of problems, but renaming and backing it up before making changes.

A complete Home Assistant backup also includes these files. A separate copy is nevertheless sensible because Home Assistant backups can become corrupted, a restore can overwrite the integration, the file may be specifically needed for later reconfiguration, and a backup should never exist only on the same system.

## Remove secrets from a published Git repository

If a JSON file was accidentally published on GitHub, normal deletion and a new commit are not enough. The file remains retrievable in Git history. At minimum, these steps are required:

1. Make the repository private immediately, if possible.
2. Remove the file from the entire Git history.
3. Take GitHub caches and forks into account.
4. Treat the token as compromised.
5. Remove the device from the Midea account and reconnect it if this generates new keys.
6. Set up the Home Assistant integration again.
7. Change the Midea account password if the credentials were also affected.

Whether pairing again actually generates a new token varies depending on the device and cloud architecture. Do not rely on changing the account password to automatically invalidate the local device token.

## Useful automations

After successful integration, the PortaSplit can be operated much more intelligently. Adjust the entity IDs to your own installation.

Cool only when windows are closed:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Switch on at high room temperature:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
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

Pre-cool before going to sleep:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
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
alias: PortaSplit bei Abwesenheit ausschalten
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
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

The desired communication direction is therefore as follows:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

## Recommended operating state

The Midea PortaSplit integrates surprisingly well with Home Assistant. After successful setup, it can be controlled locally and included in automations, eliminating a large part of cloud dependence for day-to-day operation.

From a security perspective, the integration is acceptable if some basic rules are followed: no port forwarding, keep token and key secret, encrypt backups, review debug logs before publication, secure Home Assistant, segment IoT devices, restrict outbound internet access to what is necessary, and do not install firmware and HACS updates blindly. Operated this way, the PortaSplit remains a powerful air conditioner while also becoming a sensibly integrated component of a locally controlled smart home.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: `Midea Smart AC` integration: supported device types `0xAC` and `0xCC`, PortaSplit with “Out Silent Mode”, cloud use to obtain token and key for V3 devices, manual configuration and default port 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: `Midea AC LAN` integration: supported device classes, longer TCP connection for status synchronisation and minimum Home Assistant version 2024.4.1.

3.  [midea_ac_lan: documentation of climate entities](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entities and attributes for air conditioners, including power, total energy, compressor frequency and decoding methods for energy values of individual subtypes.

4.  [midea_ac_lan: debug and configuration notes](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): device configuration stored under `/config/.storage/midea_ac_lan/`, recommendation to back up rather than delete the JSON file, and logger configuration for debug logs.

5.  [Issue 779: PortaSplit Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): request for support for the quiet mode of the outdoor unit introduced with the January 2026 firmware update, which reduces noise by around 6 decibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): manufacturer information on the security and data-protection standards EN 303 645, PSTI, NIST, GDPR and RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation and management of custom integrations that are not part of Home Assistant Core.
