---
title: "Connecting digitalSTROM with Home Assistant: local integration and automations"
navTitle: "digitalSTROM and HA"
description: "A local Home Assistant integration for the digitalSTROM server, plus motion-activated lighting, shower mode, music with 4× tapping, and automatic leaving via FRITZ!Box. Including the issues encountered in practice."
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
url: https://rafaelpfister.ch/en/blog/connecting-digitalstrom-with-home-assistant-local-integration-and-automations
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:23:27.285Z
translationReview: required
---

In digitalSTROM installations, lights, blinds, and other loads are connected to terminals in the electrical panel, controlled via push buttons and a digitalSTROM server (dSS20). When other systems such as Philips Hue, Sonos, and a FRITZ!Box are added, it makes sense to bring everything together locally in Home Assistant, without a cloud account and without storing the dSS password in Home Assistant. To do this, I wrote a small integration and published it together with suitable automations as [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under the MIT License.

In short: the dSS provides a usable JSON API with events. If used carefully, it delivers push buttons, scenes, and the Leaving and Arriving activities to Home Assistant without delay, allowing automations that digitalSTROM alone does not support.

## How the integration works

The dSS provides its API via HTTPS on port 8080, using a self-signed certificate. digitalSTROM provides app tokens for programs: the program requests a token, the user authorizes it in the Configurator under System > Access authorization, and the program then logs in with it. The dSS password remains with the user.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-s` | no progress indicator |
| `-k` | accept the dSS self-signed certificate |
| `requestApplicationToken` | creates an app token that must be authorized in the Configurator |
| `loginApplication` | exchanges the authorized app token for a session token |
| `event/subscribe` | subscribes to an event (`callScene`, `stateChange`, `buttonClick` …) under a freely chosen ID |
| `event/get` with `timeout` | waits up to 30 seconds for new events (long polling) |

</details>

The integration keeps such a long-poll connection open continuously. Every scene triggered by a push button, the app, or the Configurator arrives as an event. States such as “light on in room” or presence are stored in the dSS cache under `/usr/states` and can be queried without putting load on the terminals.

Direct queries of output values (`device/getOutputValue`) instead travel over the dS485 bus to the terminal and take half a second to a full second per value. Many such queries at short intervals can overload the meters. The integration therefore reads only blind positions and Joker outputs directly, every 15 minutes and about one minute after movement.

| Platform | Content |
|---|---|
| `light` | one light per room, switched through room scenes like the push buttons; brightness for dimmed terminals, on/off only for switched ones |
| `cover` | blinds with open, close, stop, and position |
| `scene` | moods named in the Configurator |
| `button` | Leaving (scene 72) and Arriving (scene 71) |
| `binary_sensor` | presence, wind alarm, motion sensors, Joker outputs |
| `sensor` | total consumption |

In addition, the integration reports every room scene as well as Leaving and Arriving as the `digitalstrom_local_event` event to Home Assistant. This makes it possible to freely assign 2× or 4× taps on a regular light switch.

## Automations as blueprints

The following automations are included in the repository as blueprints and can be imported into Home Assistant using the import button in the README.

### Motion-activated lighting that does not turn off manually switched-on lights

When motion is detected, the light turns on, then turns off again after an adjustable period without motion. Optionally, this applies only below a brightness threshold, only during a time window, or dimmed at night. If someone turns on the light using the push button, it remains on. A helper switch (`input_boolean`) records whether the automation turned on the light; only then does it turn it off again.

One detail only becomes apparent in operation: the “sensor clear for 2 minutes” trigger is a counter that Home Assistant discards at every restart. If the sensor last detected motion before the restart, it does not switch to “clear” again, and the light stays on. The blueprint therefore also checks every minute whether a light it turned on is still lit even though the sensor has been clear long enough.

### Turn off lights when leaving the room

The motion-lighting shutoff time is a compromise: too short, and the light turns off while someone is standing still in the room; too long, and it remains on for minutes after they leave. Another blueprint therefore uses a second sensor outside the room, typically in the hallway. If it detects motion, the sensor in the room detected motion shortly before (within 30 seconds), and it then remains clear there, the room light turns off immediately. With Hue sensors, which report “clear” about 10 seconds after the last motion, this takes around 10 to 15 seconds after leaving the room.

To ensure nobody is left sitting in the dark, the rule applies only under certain conditions: the light must have been turned on by the motion automation, shower mode must not be active, and at most one person may be home. The number of people is determined from the phones that Home Assistant knows as persons. The 30-second condition also protects against the case where someone remains still in the room while another person walks through the hallway.

### Shower mode with 2× tapping

In the shower, a motion sensor usually does not detect anyone, and after the configured time it gets dark. Tapping the bathroom button 2× triggers mood 2 (scene 17) in digitalSTROM. The automation detects this scene in the room and enables a shower mode that blocks switching off. Motion during the first 2 minutes is ignored (undressing, getting in). The first motion after that ends shower mode, after which the normal shutoff time applies again. After an adjustable maximum duration (20 minutes by default), it ends automatically.

### 4× tapping starts Sonos

If you tap a button rapidly several times, digitalSTROM cycles through moods in sequence: scene 5, 17, 18, and scene 19 on the fourth tap. If you set scene 19 to “Do not change output” for all lights in the Configurator, it is available for other purposes. A blueprint responds to this scene, turns off the room light, and starts the Sonos speaker in the room or, in rooms without a speaker, several speakers together.

The corresponding script considers three points: if a speaker is already playing, the new room is added to its group so that playback remains synchronized. If nothing can be resumed, it plays a radio station from the Radio Browser directory. And the same starting volume is set before every start; otherwise, one room plays quietly while another plays as loudly as someone last listened to music there.

### Leaving and arriving via FRITZ!Box

The FRITZ!Box Tools integration reports whether a phone is on Wi-Fi; it evaluates the box’s device list and therefore covers the 2.4 GHz network, 5 GHz network, and LAN at the same time. If nobody is home anymore and Leaving was not triggered, a blueprint triggers Leaving. On arriving home, Arriving follows and, if it is dark, a welcome light.

Additionally, an automation can turn off lights from other systems (such as Hue) and pause Sonos when Leaving is triggered. The dSS turns off the digitalSTROM lights itself.

## Floor plan as a dashboard

A clear display in Home Assistant is a floor plan using the built-in `picture-elements` card. Each room has a transparent SVG overlaid on the plan, replaced by a slightly yellow one when the light is on; tapping the room switches the light. Lights, speakers, blinds, and sensors are placed as symbols at their locations. Instructions with an example configuration are available in the repository under `docs/floor-plan-dashboard.md`.

## Issues encountered in practice

| Issue | Cause | Solution |
|---|---|---|
| Leaving button has no effect | dSS is not on the network; activities across multiple circuits run through the server | Restored the dSS network connection |
| Blinds do not respond | Wind alarm was set to “active” although no wind sensor was installed | Triggered scene 87 (“no wind”) for all rooms |
| Blinds do not raise when Leaving | Scene 72 set to “Do not change output” (`dontCare`) | `device/setSceneMode` with `dontCare=0`; the value `false` was accepted but ignored |
| Light cannot be dimmed | Fluorescent tube connected to a switched terminal (output mode 35) | The integration detects switched terminals and offers only on/off for them |
| Light remains on after restart | Counter for “clear for X minutes” is lost on restart | Additional check every minute |
| Phone is considered absent | iPhone uses a rotating private Wi-Fi address and briefly disconnects from Wi-Fi while asleep | Set Private Wi-Fi Address to “Fixed,” grace period 10 minutes |
| DECT transmits continuously | “DECT Eco” is unavailable as soon as a FRITZ! smart home device is registered | Avoid DECT outlets if DECT Eco is desired |

A device search on the Hue Bridge adds all devices currently in pairing mode. Afterwards, check the device list to ensure that only your own lights were added.

The terminals’ scene tables can be read and changed through the API, for example with `device/getSceneMode` and `device/saveScene`. Each of these calls goes over the bus. Before making changes, back up the old values, for example in a CSV file, so that you can restore them if necessary.

## Sources

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration and blueprints from this article, MIT License.

2.  [digitalSTROM: Operating and Configuration Manual](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Leaving activity (hold for 3 seconds), “Do not change output” setting in chapter 3.5.2.

3.  [digitalSTROM: Terminal Button Documentation](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Default settings for button functions by terminal type.

4.  [digitalSTROM: Operating Manuals](https://www.digitalstrom.com/bedienungsanleitungen/): Overview of manuals, planning, and installation documents.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Presence detection, Wi-Fi switches, integration options.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Basis for the floor-plan dashboard.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Importing and using blueprints.

8.  [FRITZ! Knowledge Base: FRITZ! outlet loses connection](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Notes on DECT range and radio transmission power for smart home outlets.

9.  [Radio Browser](https://www.radio-browser.info/): Free directory of radio streams, integrated in Home Assistant as a media source.
