---
title: "Home Assistant: smart home hub with a local focus"
blatt: "home-assistant"
description: "An overview of the open source smart home platform: local control instead of mandatory cloud, integrations and device connectivity, automations, installation variants, and the role of APIs, tokens, and MQTT."
fakten:
  - { label: "Project", wert: "Home Assistant (open source, Python)", href: "https://www.home-assistant.io/" }
  - { label: "Core principle", wert: "Local control; cloud only where necessary" }
  - { label: "Integrations", wert: "more than 2000, from Zigbee to vendor cloud APIs" }
  - { label: "Installation types", wert: "Home Assistant OS, Container, Core" }
  - { label: "Interfaces", wert: "REST and WebSocket API, MQTT, webhooks" }
  - { label: "Typical hardware", wert: "Raspberry Pi, mini PC, VM" }
werbung: ["newsletter"]
ctaThemen: ["smart-home-iot"]
---

# Home Assistant: smart home hub with a local focus

Home Assistant is the most widely used open source platform for home automation. Its core promise sets it apart from vendor ecosystems: the hub runs on the user's own hardware in the user's own network, devices are controlled locally wherever possible, and the data stays in the house. Cloud services are an addition, not a prerequisite.

## Architecture: hub, integrations, entities

The core is an instance that connects devices through **integrations**, one per vendor, protocol, or service. Every connected device breaks down into **entities** (a thermostat, for example, into temperature, target temperature, and mode), on which **automations** are built: triggers, conditions, actions. The connection itself follows two patterns: **locally** via standards such as Zigbee, Z-Wave, Matter, or device-specific network APIs, or **through the vendor cloud** when the device offers nothing locally. Local connections react faster and work without internet access; cloud connections depend on an account, on API availability, and on the goodwill of the vendor, whose API changes can also break integrations.

## Installation variants

**Home Assistant OS** is the complete package: a dedicated operating system with Supervisor, an add-on store, and backups, typically on a Raspberry Pi or mini PC. **Container** installations fit into existing Docker environments but do without Supervisor and add-ons. **Core**, finally, is the plain Python application for special cases. For newcomers and most households, Home Assistant OS is the documented standard path; Container is chosen by operators who maintain a self-hosting landscape anyway.

## Interfaces: tokens, MQTT, webhooks

Outward, Home Assistant speaks a **REST and WebSocket API**, secured through long-lived access tokens; it can be used to read states, call services, and connect external systems. **MQTT** serves as a message bus for devices and bridges, and **webhooks** accept events from outside. For devices with a local but undocumented API, the community does the protocol work; such integrations sometimes require device-specific keys or tokens that are obtained once from the vendor cloud.

## Operations

Home Assistant is released monthly in new versions, with breaking changes listed in the release notes; built-in **backups** (including add-ons on OS installations) preserve configuration and history. The configuration lives in a mix of YAML and UI and can be version-controlled, which means the hub follows the same operating patterns as other self-hosted services.
