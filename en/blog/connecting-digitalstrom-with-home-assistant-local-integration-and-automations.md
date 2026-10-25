---
title: "Connect digitalSTROM to Home Assistant: local integration and automations"
navTitle: "digitalSTROM and HA"
description: "A local Home Assistant integration for the digitalSTROM server, plus motion-activated lighting, shower mode, music with 4× tapping, and automatic leave detection via FRITZ!Box. Including the issues encountered in practice."
date: "2026-10-06"
kategorie: "Home Assistant and IoT"
timeToRead: "13 min read"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant
slug: "connecting-digitalstrom-with-home-assistant-local-integration-and-automations"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
translationSourceHash: 36ff960cb636859ab9f7ed1cccb3152c61e6657a074546cef23685dbde2ec8fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:47:32.735Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/connecting-digitalstrom-with-home-assistant-local-integration-and-automations
---

In digitalSTROM installations, lights, blinds, and other loads are connected to terminals in the electrical panel, controlled via pushbuttons and a digitalSTROM server (dSS20). When other systems such as Philips Hue, Sonos, and a FRITZ!Box are added, it makes sense to bring everything together locally in Home Assistant, without a cloud account and without storing the dSS password in Home Assistant. To do this, I wrote a small integration and published it together with suitable automations as [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under the MIT license.

In short: the dSS offers a usable JSON API with events. If you use it gently, pushbuttons, scenes, and the Leave and Arrive activities reach Home Assistant without delay and can trigger automations that digitalSTROM alone does not provide.

## How the integration works

The dSS provides its API over HTTPS on port 8080, using a self-signed certificate. digitalSTROM provides app tokens for programs: the program requests a token, the user approves it in the Configurator under System > Access authorization, and the program then logs in with it. The dSS password remains with the user.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Explanation of options</summary>

| Option | Effect |
|---|---|
| `-s` | no progress display |
| `-k` | accept the dSS self-signed certificate |
| `requestApplicationToken` | creates an app token that must be approved in the Configurator |
| `loginApplication` | exchanges the approved app token for a session token |
| `event/subscribe` | subscribes to an event (`callScene`, `stateChange`, `buttonClick` …) under a freely chosen ID |
| `event/get` with `timeout` | waits up to 30 seconds for new events (long poll) |

</details>

The integration keeps one such long-poll connection open continuously. Every scene triggered by a pushbutton, the app, or the Configurator arrives as an event. States such as “room light on” or presence are stored in the dSS cache under `/usr/states` and can be queried without placing load on the terminals.

Direct queries of output values (`device/getOutputValue`) instead go over the dS485 bus all the way to the terminal and take half a second to a full second per value. Many such queries at short intervals can overload the meters. The integration therefore reads only blind positions and Joker outputs directly, every 15 minutes and about one minute after movement.

| Platform | Content |
|---|---|
| `light` | one light per room, controlled through room scenes like the pushbuttons; brightness for dimmed terminals, on/off only for switched ones |
| `cover` | blinds with open, close, stop, and position |
| `scene` | moods named in the Configurator |
| `button` | Leave (scene 72) and Arrive (scene 71) |
| `binary_sensor` | presence, wind alarm, motion detectors, Joker outputs |
| `sensor` | total power consumption |

In addition, the integration reports every room scene as well as Leave and Arrive to Home Assistant as the `digitalstrom_local_event` event. This makes it possible to freely assign 2× or 4× taps on an ordinary light switch.

## Automations as blueprints

The following automations are included in the repository as blueprints and can be imported into Home Assistant using the import button in the README.

### Motion-activated lighting that does not turn off manually switched-on lights

When motion is detected, the light turns on and turns off again after a configurable period without motion. Optionally, this applies only below a brightness threshold, only during a time window, or dimmed at night. If someone turns on the light using the pushbutton, it remains on. A helper switch (`input_boolean`) remembers whether the automation turned on the light; only then does it turn it off again.

One detail only becomes apparent during operation: the “detector clear for 2 minutes” trigger is a counter that Home Assistant discards on every restart. If the detector last detected motion before the restart, there is no new transition to “clear,” and the light stays on. The blueprint therefore also checks every minute whether a light it turned on is still lit even though the detector has been clear long enough.

### Turn off lights when leaving the room

The motion-lighting turn-off time is a compromise: too short, and the light turns off while someone is standing still in the room; too long, and it remains on for minutes after they leave. Another blueprint therefore uses a second detector outside the room, typically in the hallway. If it detects motion, the detector in the room detected motion shortly beforehand (within 30 seconds), and the room then remains clear, the room light turns off immediately. With Hue detectors, which report “clear” about 10 seconds after the last motion, this is around 10 to 15 seconds after leaving the room.

To ensure nobody is left sitting in the dark, the rule applies only under certain conditions: the light must have been turned on by the motion automation, shower mode must not be active, and at most one person may be home. The number of people is derived from the phones that Home Assistant knows as persons. The 30-second condition also protects against the case where someone remains still in the room for longer while another person walks through the hallway.

### Shower mode via 2× tapping

In the shower, a motion detector usually does not detect anyone, and after the configured time it gets dark. Tapping the bathroom pushbutton 2× triggers mood 2 (scene 17) in digitalSTROM. The automation recognizes this room scene and turns on a shower mode that blocks turn-off. Motion during the first 2 minutes is ignored (undressing, getting in). The first motion after that ends shower mode, and the normal turn-off time applies again. After a configurable maximum duration (20 minutes by default), it ends on its own.

### 4× tapping starts Sonos

If you tap a pushbutton several times quickly, digitalSTROM cycles through the moods in sequence: scenes 5, 17, and 18, then scene 19 on the fourth tap. If you set scene 19 to “Do not change output” for all lights in the Configurator, it is available for other purposes. A blueprint responds to this scene, turns off the room light, and starts the Sonos speaker in the room or, in rooms without a speaker, several speakers together.

The script handles three points: if a speaker is already playing, the new room is added to its group so playback remains synchronized. If nothing can be resumed, it plays a radio station from the Radio Browser directory. And the same initial volume is set before each start; otherwise, one room plays quietly while another plays as loudly as someone last listened to music.

### Leave and Arrive via FRITZ!Box

The FRITZ!Box Tools integration reports whether a phone is on Wi-Fi; it evaluates the box’s device list and therefore covers the 2.4 GHz network, 5 GHz network, and LAN at the same time. If nobody is home anymore and Leave has not been pressed, a blueprint triggers Leave. On returning home, Arrive follows and, if it is dark, a welcome light.

In addition, an automation can turn off lights from other systems (such as Hue) and pause Sonos when leaving. The dSS turns off the digitalSTROM lights itself.

## Floor plan as a dashboard

A clear Home Assistant display is a floor plan using the built-in `picture-elements` card. Each room has a transparent SVG over the plan that is replaced with a slightly yellow one when the light is on; tapping the room switches the light. Lights, speakers, blinds, and detectors are placed as icons at their locations. Instructions with an example configuration are in the repository under `docs/floor-plan-dashboard.md`.

## Issues from real-world use

| Issue | Cause | Solution |
|---|---|---|
| Leave pushbutton has no effect | dSS is not on the network; activities across multiple electrical circuits run through the server | Restored the dSS network connection |
| Blinds do not respond | Wind alarm was set to “active” although no wind sensor is present | Triggered scene 87 (“no wind”) for all rooms |
| Blinds do not open on Leave | Scene 72 set to “Do not change output” (`dontCare`) | `device/setSceneMode` with `dontCare=0`; the value `false` was accepted but ignored |
| Light cannot be dimmed | Fluorescent tube connected to a switched terminal (output mode 35) | The integration recognizes switched terminals and offers only on/off there |
| Light remains on after restart | Counter for “clear for X minutes” is lost on restart | Additional check every minute |
| Phone is considered away | iPhone uses a changing private Wi-Fi address and briefly disconnects from Wi-Fi while asleep | Set Private Wi-Fi Address to “Fixed,” grace period of 10 minutes |
| DECT transmits continuously | “DECT Eco” is unavailable as soon as a FRITZ! Smart Home device is registered | Avoid DECT outlets if DECT Eco is desired |

A device search on the Hue Bridge adds all devices currently in pairing mode. Afterwards, check the device list to make sure only your own lights were added.

The scene tables of the terminals can be read and changed through the API, for example with `device/getSceneMode` and `device/saveScene`. Each of these calls goes over the bus. Before making changes, back up the old values, for example in a CSV file, so you can restore them if needed.

## Sources

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration and blueprints from this article, MIT license.

2.  [digitalSTROM: Operating and Configuration Manual](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Leave activity (hold for 3 seconds), “Do not change output” setting in chapter 3.5.2.

3.  [digitalSTROM: Terminal button documentation](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Factory settings for pushbutton functions by terminal type.

4.  [digitalSTROM: Operating manuals](https://www.digitalstrom.com/bedienungsanleitungen/): Overview of manuals, planning, and installation documentation.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Presence detection, Wi-Fi switches, and integration options.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Foundation of the floor-plan dashboard.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Importing and using blueprints.

8.  [FRITZ! Knowledge Base: FRITZ! outlet loses connection](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Notes on DECT range and radio transmission power for Smart Home outlets.

9.  [Radio Browser](https://www.radio-browser.info/): Free directory of radio streams, integrated in Home Assistant as a media source.
