---
title: "Connecting digitalSTROM to Home Assistant: Local Integration and Automations"
navTitle: "digitalSTROM and HA"
description: "A local Home Assistant integration for the digitalSTROM server, plus motion-activated lighting, shower mode, music with 4× taps, and automatic leaving via FRITZ!Box. Including the issues encountered in practice."
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
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:22:12.912Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/connecting-digitalstrom-with-home-assistant-local-integration-and-automations
---

In digitalSTROM installations, lights, blinds, and other loads are connected to terminals in the electrical panel and controlled via push buttons and a digitalSTROM server (dSS20). When additional systems such as Philips Hue, Sonos, and a FRITZ!Box are added, it makes sense to bring everything together locally in Home Assistant, without a cloud account and without storing the dSS password in Home Assistant. To do this, I wrote a small integration and published it together with matching automations as [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under the MIT License.

In short: The dSS provides a usable JSON API with events. If you use it gently, you can get push buttons, scenes, and the Leaving and Arriving activities into Home Assistant without delay and use them to trigger automations that digitalSTROM does not offer on its own.

## How the integration works

The dSS provides its API over HTTPS on port 8080, using a self-signed certificate. digitalSTROM provides app tokens for programs: The program requests a token, the user authorizes it in the Configurator under System > Access authorization, and the program then logs in with it. The dSS password remains with the user.

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
| `requestApplicationToken` | creates an app token that must be authorized in the Configurator |
| `loginApplication` | exchanges the authorized app token for a session token |
| `event/subscribe` | subscribes to an event (`callScene`, `stateChange`, `buttonClick` …) under a freely chosen ID |
| `event/get` with `timeout` | waits up to 30 seconds for new events (long poll) |

</details>

The integration keeps one such long-poll connection open continuously. Every scene triggered by a push button, the app, or the Configurator arrives as an event. States such as “room light on” or presence are stored in the dSS cache under `/usr/states` and can be queried without placing a load on the terminals.

Direct queries of output values (`device/getOutputValue`) instead travel via the dS485 bus to the terminal and take half a second to a full second per value. Many such queries at short intervals can overload the meters. The integration therefore reads only blind positions and Joker outputs directly, every 15 minutes and about one minute after movement.

| Platform | Content |
|---|---|
| `light` | one light per room, switched via room scenes like the push buttons; brightness for dimmed terminals, on/off only for switched ones |
| `cover` | blinds with open, close, stop, and position |
| `scene` | moods named in the Configurator |
| `button` | Leaving (scene 72) and Arriving (scene 71) |
| `binary_sensor` | presence, wind alarm, motion sensors, Joker outputs |
| `sensor` | total consumption |

In addition, the integration reports every room scene as well as Leaving and Arriving to Home Assistant as the `digitalstrom_local_event` event. This lets you freely assign actions to 2× or 4× taps on an ordinary light switch.

## Automations as blueprints

The following automations are included in the repository as blueprints and can be imported into Home Assistant using the import button in the README.

### Motion-activated light that does not turn off manually switched-on lights

When motion is detected, the light turns on and turns off again after an adjustable period without motion. Optionally, this applies only below a brightness threshold, only within a time window, or dimmed at night. If someone turns on the light using the push button, it stays on. A helper switch (`input_boolean`) remembers whether the automation turned on the light; only then does it turn it off again.

One detail only becomes apparent in operation: The “sensor clear for 2 minutes” trigger is a timer that Home Assistant discards on every restart. If the sensor last detected motion before the restart, it will not switch to “clear” again, and the light remains on. The blueprint therefore also checks every minute whether a light it turned on is still lit even though the sensor has been clear long enough.

### Turn off the light when leaving the room

The motion light's switch-off time is a compromise: Too short, and the light turns off while someone is standing still in the room; too long, and it stays on for minutes after they leave. Another blueprint therefore uses a second sensor outside the room, typically in the hallway. If it detects motion, the sensor in the room detected motion shortly beforehand (within 30 seconds), and the room then remains clear, the room light turns off immediately. With Hue sensors, which report “clear” about 10 seconds after the last motion, this is around 10 to 15 seconds after stepping out.

To ensure nobody is left in the dark, the rule applies only under certain conditions: The light must have been turned on by the motion automation, shower mode must not be active, and at most one person may be home. The number of people is determined from the phones Home Assistant recognizes as persons. The 30-second condition also protects against the case where someone is still in the room while another person walks through the hallway.

### Shower mode with 2× taps

In the shower, a motion sensor usually detects nobody, and it gets dark after the configured time. Tapping the bathroom push button 2× triggers mood 2 (scene 17) in digitalSTROM. The automation detects this scene in the room and enables a shower mode that prevents the light from turning off. Motion during the first 2 minutes is ignored (undressing, getting in). The first motion after that ends shower mode, and the normal switch-off time applies again. After an adjustable maximum duration (20 minutes by default), it ends automatically.

### 4× taps starts Sonos

If you tap a push button several times quickly, digitalSTROM cycles through the moods in sequence: scene 5, 17, 18, and 19 on the fourth tap. If you set scene 19 to “Do not change output” for all lamps in the Configurator, it is available for other purposes. A blueprint responds to this scene, turns off the room light, and starts the Sonos speaker in the room or, in rooms without a speaker, several speakers together.

The script handles three points: If a speaker is already playing, the new room is added to its group so playback remains synchronized. If nothing can be resumed, it plays a radio station from the Radio Browser directory. And the same starting volume is set before every start; otherwise, one room plays quietly while another plays as loudly as someone last listened to music.

### Leaving and arriving via FRITZ!Box

The FRITZ!Box Tools integration reports whether a phone is on Wi-Fi; it evaluates the box's device list and therefore covers the 2.4 GHz network, 5 GHz network, and LAN at the same time. If nobody is home anymore and Leaving has not been pressed, a blueprint triggers Leaving. On returning home, Arriving follows and, if it is dark, a welcome light.

In addition, an automation can turn off lights from other systems (such as Hue) and pause Sonos when leaving. The dSS turns off the digitalSTROM lights itself.

## Floor plan as a dashboard

A clear Home Assistant display is a floor plan using the built-in `picture-elements` card. Each room has a transparent SVG over the plan, which is replaced with a slightly yellow one when the light is on; tapping the room switches the light. Lamps, speakers, blinds, and sensors appear as icons at their locations. The instructions with an example configuration are in the repository under `docs/floor-plan-dashboard.md`.

## Issues encountered in practice

| Issue | Cause | Solution |
|---|---|---|
| Leaving button has no effect | dSS is not on the network; activities across multiple electrical circuits run through the server | Restored the dSS network connection |
| Blinds do not respond | Wind alarm was set to “active” even though no wind sensor exists | Triggered scene 87 (“no wind”) for all rooms |
| Blinds do not open when leaving | Scene 72 set to “Do not change output” (`dontCare`) | `device/setSceneMode` with `dontCare=0`; the value `false` was accepted but ignored |
| Light cannot be dimmed | Fluorescent tube connected to a switched terminal (output mode 35) | Integration detects switched terminals and offers on/off only there |
| Light remains on after restart | Timer for “clear for X minutes” is lost on restart | Additional check every minute |
| Phone is considered away | iPhone uses a rotating private Wi-Fi address and briefly disconnects from Wi-Fi while asleep | Set Private Wi-Fi Address to “Fixed,” grace period of 10 minutes |
| DECT transmits continuously | “DECT Eco” is unavailable whenever a FRITZ! smart home device is registered | Do without DECT outlets if DECT Eco is desired |

A device search on the Hue Bridge adds all devices currently in pairing mode. Afterwards, check the device list to ensure that only your own lamps were added.

The terminals' scene tables can be read and changed through the API, for example with `device/getSceneMode` and `device/saveScene`. Each of these calls goes over the bus. Before making changes, back up the old values, for example in a CSV file, so you can restore them if needed.

## Changelog

- **October 25, 2026:** Updated the source link to the new address of the FRITZ! knowledge base.

## Sources

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration and blueprints from this article, MIT License.

2.  [digitalSTROM: Operating and Configuration Manual](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Leaving activity (hold for 3 seconds), “Do not change output” setting in section 3.5.2.

3.  [digitalSTROM: Terminal Button Documentation](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Default settings of button functions by terminal type.

4.  [digitalSTROM: Operating Instructions](https://www.digitalstrom.com/bedienungsanleitungen/): Overview of manuals, planning, and installation documentation.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Presence detection, Wi-Fi switches, integration options.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Foundation of the floor plan dashboard.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Importing and using blueprints.

8.  [FRITZ! Knowledge Base: FRITZ! outlet loses connection](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Information on DECT range and radio transmission power for smart home outlets.

9.  [Radio Browser](https://www.radio-browser.info/): Open directory of radio streams, integrated into Home Assistant as a media source.
