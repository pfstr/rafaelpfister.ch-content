---
title: "Control the Midea PortaSplit locally with Home Assistant and operate it securely"
navTitle: "Set up PortaSplit"
description: "From the right community integration to an IoT VLAN: how to set up the PortaSplit, protect tokens and keys, and restrict cloud and network access."
date: "2026-07-24"
kategorie: "Smart Home & IoT"
timeToRead: "14 min read"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-integration"
  - "serverless-newsletter-cloudflare-workers-d1"
translationOf: "midea-portasplit-home-assistant-einrichten"
slug: "midea-portasplit-home-assistant-setup-and-hardening"
url: "https://rafaelpfister.ch/en/blog/midea-portasplit-home-assistant-setup-and-hardening"
---

The Midea PortaSplit can be controlled directly on the local network via Home Assistant after setup. To do this, the community integration requires two device-specific credentials from the Midea cloud: a token and a key.

This article covers selecting, setting up and securing the integration. The solutions described come from the community and are not officially supported by either Midea or Home Assistant. Firmware or cloud changes may therefore affect their behaviour at any time. Background information on the token interface and the ambiguous shutdown warning can be found in the [analysis of the Midea Cloud APIs](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## How local control works

Once configured, the actual control commands are sent directly from Home Assistant to the PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

A switching command does not need to pass through an external Midea server, response times are short, an outage of the Midea cloud does not necessarily disable local control that has already been configured, and the device remains controllable without internet access in principle.

However, newer devices using the so-called V3 protocol do not accept local commands without protection. Home Assistant needs two device-specific values, a token and a key, which are used to authenticate and encrypt the local connection. During initial setup, the integration retrieves them once via a Midea cloud interface and then stores them locally; no cloud connection is required for subsequent control.

In simplified terms, the process is as follows:

1. The PortaSplit is connected to MSmartHome.
2. Home Assistant signs in to a Midea cloud.
3. Home Assistant receives the device ID, token and key.
4. The token and key are stored locally.
5. Home Assistant controls the PortaSplit directly on the LAN.

## Which integration is suitable

### Midea Smart AC

The repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> focuses on Midea air conditioners and related OEM models and supports the device types `0xAC` and `0xCC`. It offers local control, graphical setup, automatic discovery, manual setup with token and key, and automatic querying of device capabilities. The PortaSplit's “Out Silent Mode” is explicitly supported.

As an indication of compatibility, the project lists the Artic King, Midea Air, NetHome Plus, SmartHome or MSmartHome, Toshiba AC NA and 美的美居 apps, among others. The PortaSplit typically uses MSmartHome in Europe and therefore fits into this ecosystem.

### Midea AC LAN

The repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> supports not only air conditioners but many other Midea device classes: dehumidifiers, fans, air purifiers, washing machines, tumble dryers, dishwashers, water heaters, heat pumps, refrigerators and more, in some cases also under third-party brands such as Carrier or Electrolux. It also provides local communication, automatic device discovery and additional sensors and, according to the project description, keeps a longer TCP connection open to the device in order to synchronise status changes promptly. Home Assistant 2024.4.1 or later is required.

The main drawback at present is the developer's warning: the cloud token APIs used to add new devices are being gradually shut down. This may make it impossible to add new devices later.

### Recommendation

For a PortaSplit-only installation, I would start with `Midea Smart AC` and keep `Midea AC LAN` in mind as an alternative. `Midea Smart AC` is more tightly focused on air conditioners and explicitly documents the current PortaSplit features.

Running both integrations simultaneously and permanently with the same device is not advisable. Multiple parallel connections lead to status issues, unnecessary network traffic and behaviour that is difficult to trace.

## What the integration provides

After setup, the PortaSplit appears as a `climate` entity in Home Assistant. Depending on the firmware and integration, the following functions are available, among others:

- Switching on and off
- Setting the target temperature
- Reading the current room temperature
- Cooling, dehumidification and fan-only operation
- Setting fan speed
- Controlling the swing function
- Eco and boost modes
- Reading humidity
- Displaying error codes
- Reading energy and power values
- Displaying compressor values
- Activating quiet mode for the outdoor unit

Which entities actually appear depends on the model, firmware, protocol used and respective integration. `Midea Smart AC` queries the capabilities reported by the device and hides functions the model does not support. `Midea AC LAN` also documents extensive climate entities, including temperature, humidity, current power, total energy, compressor frequency, pump status and various operating modes, and specifies dedicated methods for decoding energy data for certain PortaSplit subtypes.

Not every displayed measurement is necessarily correct. Energy consumption and power in particular are transmitted in different formats by different Midea models. If Home Assistant shows obviously incorrect values, the decoding method in use usually needs adjusting rather than the device being faulty.

## Requirements

You need a Midea PortaSplit with Wi-Fi capability, a 2.4 GHz Wi-Fi network, the MSmartHome app, a Midea user account, Home Assistant, HACS and network access between Home Assistant and the PortaSplit. The PortaSplit should first be connected normally using the MSmartHome app, and only then added to Home Assistant.

## Step 1: Connect the PortaSplit to MSmartHome

1. Install the MSmartHome app.
2. Create a Midea account or sign in.
3. Put the PortaSplit into Wi-Fi pairing mode.
4. Connect the device to the 2.4 GHz Wi-Fi network.
5. Check that the PortaSplit can be controlled through the app.

Many IoT devices still support only 2.4 GHz. If the router uses the same SSID for 2.4 and 5 GHz, setup will usually still work. If there are problems, it can help to provide a separate 2.4 GHz Wi-Fi network temporarily.

## Step 2: Install HACS

HACS is the Home Assistant Community Store. It enables the installation of community integrations that are not part of Home Assistant Core. After installing HACS, open HACS, go to Integrations, search for `Midea Smart AC`, download the integration and restart Home Assistant. Alternatively, search for `Midea AC LAN`.

HACS simplifies installation and updates. However, it does not turn a custom integration into an officially reviewed Home Assistant component. This distinction is important from a security perspective and is discussed below.

## Step 3: Add Midea Smart AC

After restarting, go to Settings, Devices & services and Add integration, then search for `Midea Smart AC` and then `Discover devices`. The integration can either scan the entire local network or query the PortaSplit's IP address directly.

If the device is found, the integration requires the region, Midea account, password and device ID for newer V3 devices, as well as the token and key derived from these. The cloud region must match the account being used. If there are problems, the project recommends trying the other available regions as well.

### Manual setup

If automatic setup fails, the device can be configured manually. For `Midea Smart AC`, the following details are required:

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

For V3 devices, the documentation specifies the token as a 128-character hexadecimal string and the key as a 64-character hexadecimal string. Both values are secrets and must be treated accordingly. Those who do not want to obtain the credentials through discovery can retrieve them with their own account using the CLI `msmart-ng`.

## Operating the PortaSplit securely

Controlling the PortaSplit locally brings some control back from the manufacturer's cloud, but it also shifts responsibility to your own network. The following measures ensure that a mishap involving the token and key causes limited damage and that the device remains properly isolated.

### Token and key are secrets

The token and key authenticate local communication with the device and must be treated like a password. Most importantly, they do not belong in logs, unencrypted backups or a repository.

### No port forwarding to the PortaSplit

The most common avoidable mistake would be making the local device port directly accessible from the internet. A rule such as this would be dangerous:

```text
Internet → TCP 6444 → PortaSplit
```

There is no good reason to make the PortaSplit directly accessible from the internet. Home Assistant is already on the local network and serves as the controlling instance. The router should have no port forwarding to the PortaSplit, UPnP should be restricted or disabled where possible, incoming connections should be blocked by default, and no DMZ exemption should be used for the device.

### Separate IoT VLAN

The best network architecture is a separate IoT network:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

The PortaSplit is located in the IoT VLAN. Home Assistant is allowed targeted access to the device, but the PortaSplit must not be able to access PCs, NAS devices and other internal systems without restriction. A possible firewall policy:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

During initial setup, the device needs internet access to the Midea cloud. Once local setup has succeeded, you can test whether outgoing internet access can be blocked. Do not set a final block immediately. First check whether local control continues to work, whether the device remains accessible after a restart, whether it survives a router restart, whether it still responds after several days, whether the MSmartHome app is still needed, and whether firmware updates are still offered. If you want to continue using the cloud and firmware updates, you can allow outgoing internet access temporarily and block it again afterwards.

### Network segmentation can prevent discovery

Automatic device discovery often relies on broadcast or multicast traffic, which is normally not routed across VLAN boundaries. Home Assistant may therefore not find the PortaSplit automatically even if a normal IP connection would be allowed.

In that case, it can help to set up the PortaSplit temporarily in the same VLAN as Home Assistant, specify the device IP manually, use a suitable broadcast relay function, or define targeted firewall rules after setup. From a security perspective, manual configuration is often even the better option because it does not require additional broadcast traffic to be allowed between networks.

### Static DHCP assignment

The PortaSplit should receive a fixed DHCP assignment in the router:

```text
PortaSplit → 192.168.30.25
```

A DHCP reservation is usually preferable to a static IP configured on the device. Home Assistant can find the device reliably, firewall rules can be restricted to a fixed address, troubleshooting becomes easier, and the assignment remains stable after router or device restarts. A firewall rule can therefore be formulated very narrowly:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

The port actually required must be verified against the integration and your own device.

### Home Assistant as the central trust anchor

Those who control the PortaSplit locally shift some trust from the Midea cloud to Home Assistant. If Home Assistant is compromised, an attacker may be able to control not only the air conditioner but the entire smart home.

Home Assistant should therefore be updated regularly, not exposed through unprotected port forwarding, protected with a strong, unique password, use multi-factor authentication, create encrypted backups, contain only necessary add-ons, and not allow unnecessary SSH access from the internet. For remote access, a VPN, Home Assistant Cloud or a properly configured reverse proxy are better options than simple port forwarding on port 8123.

### HACS and supply-chain risk

`Midea Smart AC` and `Midea AC LAN` are custom integrations. They run inside Home Assistant and therefore have extensive access to its runtime environment. A malicious or compromised integration could theoretically read configuration data, extract secrets, establish network connections, scan devices on the local network, read the states of other entities, transfer data to external systems and affect Home Assistant availability.

This does not mean that the integrations mentioned are malicious. Both projects are publicly viewable, actively developed and have a visible community. However, open source is not an automatic security guarantee. Before installation, it is worth checking at least whether the repository is actively maintained, whether there are regular releases, how many people contribute to the code, whether there are open security issues, whether maintainers or repository owners have recently changed, whether HACS points to the expected repository, and whether an update contains unusually large or unexplained changes.

Updates should not be installed blindly immediately after release. Particularly for security-critical smart-home systems, it is sensible to wait a few days and review release notes and reported issues.

### Secure the cloud account

As long as the Midea cloud is used for setup or app control, the Midea account remains part of the security model. It should have a unique password that is not shared with other services, a password manager, multi-factor authentication if offered, removal of old smartphones and sessions, no shared accounts, and regular checks of which devices are registered in the account.

If the Home Assistant integration requests a username and password during setup, check whether the credentials are used only for the one-off token retrieval or stored permanently. The developers of `Midea Smart AC` state that devices are not linked to built-in integration accounts after setup and that the token and key can also be obtained manually through the CLI using your own account. Where possible, your own account is preferable to third-party or integrated shared accounts.

### Block the cloud or not?

After successful setup, the question arises whether the PortaSplit's internet access should be blocked completely. Arguments for blocking it include less telemetry, lower dependence on external services, a smaller attack surface through the manufacturer's cloud, the fact that the device cannot contact arbitrary external destinations, and reduced impact from cloud-side changes.

Arguments against it include that the MSmartHome app may no longer work outside the home network, firmware updates may no longer download, time or cloud functions may fail, signing in again or restoring the device may become more difficult, and some devices may behave unexpectedly after being offline for a long time.

A pragmatic sequence is: set up the device normally, test Home Assistant and the app, back up the token and configuration, block internet access, restart the device and Home Assistant, observe for several days, and if necessary only restore internet access temporarily.

### Firmware updates: security gain or integration risk?

Firmware updates are a dilemma for IoT devices. They can fix known vulnerabilities, improve stability, modernise security mechanisms and add new features. But they can also alter local interfaces, break reverse-engineered integrations, invalidate tokens, disable the local API and introduce new cloud dependencies.

For example, the PortaSplit firmware released in January 2026 introduced a new quiet mode for the outdoor unit, reducing noise by around 6 decibels. The community integrations first had to analyse and implement it, documented in a dedicated GitHub issue for the PortaSplit.

The conclusion is: do not prevent firmware updates categorically; before an update, check whether other Home Assistant users report problems, back up the configuration and token beforehand, create a Home Assistant backup, and test local control fully after the update. Security does not mean “never update”. Outdated firmware can be more dangerous than a temporarily incompatible integration.

### Debug logs contain sensitive data

When problems occur, open-source projects often request debug logs. The documentation for `Midea AC LAN` shows how to enable logging for the two relevant components:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

The logs can then be downloaded via Settings, System and Logs. Depending on the integration and error, such logs may contain local IP addresses, device ID, serial number, model identifier, cloud responses, account information, tokens or parts of them, network packets, as well as timestamps and usage patterns. They must therefore be reviewed and sensitive values redacted before uploading them to a public GitHub issue.

Once troubleshooting is complete, debug logging should be removed again. Permanently enabled debug logging not only increases storage usage, it also increases the amount of sensitive information in backups.

### What Midea itself says about security

Midea promotes its SmartHome ecosystem as being aligned with several security and privacy standards, including EN 303 645, UK PSTI, NIST, GDPR-compliant data processing and the requirements of the EU Radio Equipment Directive. These are positive signals, but they do not say how every individual PortaSplit firmware version, cloud endpoint and local API is actually implemented. Certification and marketing claims do not replace technical assessment of the specific device.

Likewise, it would be wrong to infer from a community integration's warning that the PortaSplit is generally insecure. The issue described concerns the architecture of long-lived tokens and their use by unofficial clients.

### Risk by scenario

| Scenario | Risk | Reasoning |
| --- | --- | --- |
| Normal home network without port forwarding | manageable | An attacker must first gain access to Wi-Fi, Home Assistant or a backup. |
| Flat home network with many insecure IoT devices | medium | A compromised IoT device can reach the PortaSplit or Home Assistant on the same network. |
| PortaSplit directly accessible from the internet | high | The device should never be exposed through port forwarding. |
| Token and key publicly available on GitHub | high | The secrets must be considered compromised; whether they can be revoked is not guaranteed. |
| Separate IoT VLAN, restrictive firewall, local control | comparatively low | Even if the device has a vulnerability, its ability to move around the network is severely limited. |

## Backing up the configuration

Backing up the token, key and configuration is the most important one-off step: once the cloud token interfaces are closed, a backup is the only way to set up the device again. `Midea AC LAN` stores a JSON configuration file for V3 devices after successful setup. The documented path is:

```text
/config/.storage/midea_ac_lan/
```

The file uses the device ID as its filename:

```text
<device-id>.json
```

This file is not an ordinary text note. It may contain the device ID, serial number, IP address, token, key, protocol information, as well as cloud and device parameters. Accordingly:

- Do not upload it to a public GitHub repository.
- Do not post it in forums.
- Do not share it as an unredacted screenshot.
- Do not send it by unencrypted email.

Even a private Git repository is not automatically the right storage location because secrets remain in Git history, even if they are later removed from the current file. Better options are an encrypted backup, a password manager with a file attachment, an encrypted NAS backup, encrypted offline storage or an encrypted archive with the password stored separately.

To back up via the Home Assistant terminal:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Display the file:

```bash
cat <device-id>.json
```

The file should not be transferred through a public web service for copying. An encrypted archive is better, which can then be placed in an encrypted backup:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

The files in `.storage` should not be edited manually. The developer expressly recommends neither deleting nor changing the JSON file directly if problems occur, but renaming and backing it up before making changes.

A complete Home Assistant backup also includes these files. A separate copy is still sensible because Home Assistant backups can become corrupted, a restore can overwrite the integration, the file may be needed specifically for a future reconfiguration, and a backup should never exist only on the same system.

## Removing secrets from a published Git repository

If a JSON file has accidentally been published on GitHub, normal deletion and a new commit are not enough. The file remains retrievable in Git history. At minimum, the following steps are required:

1. Make the repository private immediately, if possible.
2. Remove the file from the entire Git history.
3. Take GitHub caches and forks into account.
4. Treat the token as compromised.
5. Remove the device from the Midea account and reconnect it if this generates new keys.
6. Set up the Home Assistant integration again.
7. Change the Midea account password if login credentials were also affected.

Whether pairing again actually generates a new token varies depending on the device and cloud architecture. You should not rely on changing the account password automatically invalidating the local device token.

## Useful automations

After successful integration, the PortaSplit can be operated much more intelligently. Adapt the entity IDs to your own installation.

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

Switch on when the room temperature is high:

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

Pre-cool before bedtime:

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

The Midea PortaSplit integrates surprisingly well with Home Assistant. Once setup is complete, it can be controlled locally and included in automations, removing a large part of the cloud dependency for everyday operation.

From a security perspective, the integration is acceptable if a few basic rules are followed: no port forwarding, keep the token and key secret, encrypt backups, review debug logs before publication, secure Home Assistant, segment IoT devices, restrict outgoing internet access to what is necessary, and do not install firmware and HACS updates blindly. Used this way, the PortaSplit remains a powerful air conditioner while also becoming a sensibly integrated part of a locally controlled smart home.

## Sources

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: supported device types `0xAC` and `0xCC`, PortaSplit with “Out Silent Mode”, cloud use to obtain the token and key for V3 devices, manual configuration and default port 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN`: supported device classes, longer TCP connection for status synchronisation and minimum Home Assistant version 2024.4.1.

3.  [midea_ac_lan: documentation for climate entities](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entities and attributes for air-conditioning devices, including power, total energy, compressor frequency and the decoding methods for energy values of individual subtypes.

4.  [midea_ac_lan: debug and configuration notes](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): storage of device configuration under `/config/.storage/midea_ac_lan/`, recommendation to back up rather than delete the JSON file, and logger configuration for debug logs.

5.  [Issue 779: PortaSplit Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): request to support the quiet mode for the outdoor unit introduced with the January 2026 firmware update, which reduces noise by around 6 decibels.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): manufacturer information on the security and privacy standards EN 303 645, PSTI, NIST, GDPR and RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation and management of custom integrations that are not part of Home Assistant Core.
