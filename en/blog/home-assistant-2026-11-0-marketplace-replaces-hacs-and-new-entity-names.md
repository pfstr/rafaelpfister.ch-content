---
title: "Home Assistant 2026.11.0: Marketplace Replaces HACS and New Entity Names"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 adopts HACS as an integrated Marketplace, rebuilds entity names from the device and entity, and introduces numerous backward-incompatible changes. Before updating, check dashboards, templates, and presence detection via the FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant and IoT"
timeToRead: "6 min read"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-replaces-hacs-and-new-entity-names"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/en/blog/home-assistant-2026-11-0-marketplace-replaces-hacs-and-new-entity-names
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:23:45.629Z
translationReview: required
---

# Home Assistant 2026.11.0: Marketplace Replaces HACS and New Entity Names

Home Assistant 2026.11.0 was released on November 4, 2026, following 2026.10.4. The release integrates the former Home Assistant Community Store (HACS) directly into Home Assistant as the Marketplace, changes how entity names are formed, and includes a long list of backward-incompatible changes, including changes to Device Trackers from FRITZ!Box Tools and authentication through non-local addresses. According to the release blog post, 38 new integrations have been added. The release notes do not mention a security fix with a CVE. The release is aimed at all Home Assistant operators, especially installations with Custom Integrations from HACS, such as those used in the articles about [Midea PortaSplit](/blog/midea-portasplit-home-assistant) and [digitalSTROM](/blog/digitalstrom-home-assistant).

## The Versions at a Glance

| Version | Date | Type | Key points |
|---|---|---|---|
| 2026.11.0 | 11/04/2026 | Monthly release | HACS becomes the Marketplace, new entity names, Device Trackers from network scanners always have a device, authentication check for non-local addresses, no more `deps` folder, 38 new integrations |

## Marketplace: HACS Becomes Part of Home Assistant

With 2026.11, HACS becomes part of Home Assistant and is now called Marketplace, according to the release blog post. Administrators can find it under Settings > Marketplace and install integrations, dashboard cards, themes, and templates there; a GitHub account is not required. After installing an integration, no restart is necessary, but updates still require a restart.

Home Assistant migrates existing HACS installations on the first startup after the update. Installed projects remain installed and continue to receive updates; Custom Repositories and update entities are migrated as well. According to the vendor, no action is required. The HACS installer ends with Home Assistant 2026.11, a separate HACS installation is no longer possible on this version, and the previous HACS will receive no new releases. In the future, report issues in the Home Assistant issue tracker.

When first opened, the Marketplace displays a notice that community projects are neither tested, reviewed, nor supported by the Home Assistant project. You must confirm this notice before installing or updating. Projects outside the catalog can be added through “Add from link”; this requires a GitHub connection via Device Sign-in that reads public information only.

For the Midea PortaSplit, this affects the Custom Integration `Midea AC LAN` and the `apexcharts-card` card if both are installed through HACS. According to the migration information, they remain installed and continue to receive updates. The supply-chain risk of Custom Integrations described in the article on [securing the PortaSplit](/blog/midea-portasplit-home-assistant-absichern) remains with the Marketplace, as the vendor’s risk notice shows. After the update, check under Settings > Marketplace whether all projects and Custom Repositories previously installed through HACS are present.

## New Entity Names

Starting with 2026.11, entity names consist of the name of the entity and its context, usually the device name. Names assigned by users are migrated according to the same rules, and the old name remains as an alias. Cards, badges, and rows without their own name now display the device name followed by the entity name instead of the Friendly Name. According to the release notes, entity IDs, automations, and scripts are not affected.

For templates, the vendor recommends the functions `entity_name()`, `device_name()` and `area_name()`; the `friendly_name` attribute is planned to be marked as deprecated later. After the update, check your dashboards for changed labels and adjust templates that evaluate `friendly_name`.

## FRITZ!Box Tools and Device Trackers

Network scanner trackers, including FRITZ!Box Tools, UniFi Network, NETGEAR, and Nmap Tracker, now always have a device. Renamed tracker entities receive the device name at the beginning of their name. New trackers from these integrations are always created disabled; previously, they were active if another integration already knew the device’s MAC address. Existing trackers retain their state, whether enabled or disabled.

For presence automations through the FRITZ!Box, such as the automatic leave detection described in the [digitalSTROM article](/blog/digitalstrom-home-assistant), this means that existing trackers continue to work. However, you must first enable the tracker for a newly added smartphone before it provides a state in automations.

The changelog also lists several fixes for FRITZ!Box Tools, including creating Wi-Fi switches, filtering FRITZ!Box hosts without an IP address from Device Trackers, a reload loop for devices without traffic data, and a clearer error message for invalid authentication in the setup dialog. The FRITZ!Box Call Monitor integration now reports a configuration error when permissions are missing.

## Midea and digitalSTROM

The Midea improvements in the release blog post concern the official integration `Midea`, which has been part of Core since Home Assistant 2026.8. The Custom Integration `Midea AC LAN` (domain `midea_ac_lan`) is separate from it. New features include controls and a progress sensor for washing machines and dryers, door sensors for refrigerators, water-heater entities, an on/off switch for air purifiers, and a switch for the sound signal on dehumidifiers. For air conditioners, the changelog mentions a `reset_filter` button that can mark the filter as cleaned. The integration has reached Platinum quality scale. According to the integration page, control and status queries run locally; the cloud is only needed for the token and key of V3 devices.

The 2026.11.0 changelog contains no entry for digitalSTROM. The integration described in the [digitalSTROM article](/blog/digitalstrom-home-assistant) is a Custom Integration; if it is included through HACS as a Custom Repository, the described migration to the Marketplace applies to it.

## Additional Backward-Incompatible Changes

The full list is available in the release blog post. These points are particularly relevant for operation:

- **Auth:** When accessing through a non-local IP address, the scheme, host, and port in the browser must match the address Home Assistant sees, otherwise login fails with “Invalid client id”. Due to a bug, the previous check accepted such addresses as soon as they included a port. In this case, add your reverse proxy to the Trusted Proxies of the HTTP integration or use a domain name.
- **Python dependencies:** Home Assistant no longer installs packages in the `deps` folder in the configuration directory. Only Core installations outside a virtual environment are affected; OS, Supervised, and Container installations are not. After the update, the `deps` folder can be deleted.
- **SMTP:** An unreadable attachment now results in an error; previously, the message was sent without the attachment.
- **Telegram:** The old `notify` platform is removed. Use the Telegram bot’s Notify entities and the `notify.send_message` action.
- **Z-Wave:** zwave-js-server 3.11.0 or later (schema 51), the Z-Wave JS app from 1.9.0, or Z-Wave JS UI from v11.24.2 are required.
- **Uptime Kuma:** Versions before 2.0.0 are no longer supported.
- **Philips Hue:** The `"None"` light effect is removed; `"off"` applies instead.
- **Matter and Electra Smart:** `climate.turn_on` resumes the most recently active mode. To use a specific mode, use `climate.set_hvac_mode`.
- **Polling interval:** For Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato, and Risco, a custom interval is ignored and the default is used. Control4 always polls every 5 seconds; for a different interval, disable polling and call `homeassistant.update_entity` from an automation.

The Flipr integration was removed because the company behind it was liquidated.

## New Features

In addition to the Marketplace, the release blog post mentions excluding individual entities, devices, areas, floors, or labels from a target; copying and pasting dashboard sections; a date card; and an alert card with a visual editor. The Energy Dashboard can derive an energy sensor from a power sensor. For troubleshooting, there is a “Triggered by” filter in activity, live logs per integration, and the ability to delete statistics for a period. Backups and media can be stored on local drives. Newly supported are Matter 1.5 cameras with WebRTC as well as LoRaWAN (ChirpStack, The Things Network, SenseCAP) and Zigbee Green Power.

For the alert card, the `color` option is now called `alert_color`; you only need to adjust dashboards you created in YAML yourself. The new Debug console integration executes Python code inside Home Assistant, has full access, and must therefore be enabled in `configuration.yaml`; only administrators can use it.

## Recommended Approach

The target version is 2026.11.0. Before updating, read the backward-incompatible changes in the release blog post for the integrations you use. After the update, check the Marketplace to ensure all previous HACS projects were migrated, review dashboards and templates for the new entity names, and enable newly created Device Trackers as needed. If access is through a reverse proxy, test logging in. As of November 5, 2026, the security advisories in the GitHub repository for Home Assistant Core contain no entry assigned to 2026.11.0; the release blog post does not include information about Known Issues.

## Sources

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Release blog post with the Marketplace, entity names, new integrations, and backward-incompatible changes.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Complete changelog, including entries for FRITZ!Box Tools, FRITZ!Box Call Monitor, and Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Release tag published on November 4, 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Documentation for the official Midea integration, introduced in 2026.8, with local control.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Published security advisories, as of November 5, 2026, without an entry for 2026.11.0.

6.  [Midea PortaSplit in Home Assistant: Setup and Dashboard](/blog/midea-portasplit-home-assistant): Installation of Midea AC LAN and apexcharts-card through HACS.

7.  [Securing Midea PortaSplit in Home Assistant](/blog/midea-portasplit-home-assistant-absichern): Supply-chain risk of Custom Integrations.

8.  [Connecting digitalSTROM to Home Assistant](/blog/digitalstrom-home-assistant): Local digitalSTROM integration and presence detection through the FRITZ!Box.
