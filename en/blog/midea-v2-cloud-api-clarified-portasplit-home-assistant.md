---
title: "Midea V2, V3, and Cloud API: What This Actually Means for the PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "Local device protocol, private app endpoints, and the official partner API use similar version names. This source analysis separates these layers and puts the shutdown warning into context."
date: "2026-07-25"
kategorie: "Home Assistant and IoT"
timeToRead: "11 min read"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
slug: "midea-v2-cloud-api-clarified-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:29:31.846Z
translationReview: automatic
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/midea-v2-cloud-api-clarified-portasplit-home-assistant
---

In the context of the Midea PortaSplit, “V2” refers to several independent things. There is a local V2 device protocol, version numbers in private app endpoints, and an official cloud-to-cloud API V2 for partners. Equating these layers inevitably leads to incorrect conclusions about local control.

The project `Midea AC LAN` warns in its [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) that existing token interfaces would be closed and replaced by a cloud-based V2 API. A review of the discussions, current code, and official Midea documentation paints a more nuanced picture:

> An official Midea cloud-to-cloud API V2 exists. However, it is not identical to the token interface used by Home Assistant, nor is it the local V2 or V3 device protocol. No officially announced shutdown of local PortaSplit control with a specific date is documented. In June 2026, it was also demonstrated that the supposedly shut-down SmartHome token API was still working—the community library’s previous request was simply incomplete.

This is part 3 of the series; [part 1](/blog/midea-portasplit-home-assistant) covers setup through to the dashboard, while [part 2](/blog/midea-portasplit-home-assistant-absichern) covers securing tokens, keys, and the home network. This article is current as of October 25, 2026.

![Home Assistant dashboard for the Midea PortaSplit in cooling mode: metrics at the top, thermostat set to 22 °C, charts for room temperature, power consumption, daily energy, compressor frequency, compressor operation, and fan speed, followed by technical values and status.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Why the previous assessment needs to be corrected

In an earlier version of the [article on tokens and keys](/blog/midea-portasplit-home-assistant-absichern), I paraphrased the warning from project `Midea AC LAN` as an announced shutdown of the cloud interfaces. That reflected the wording of the project README, but it was too strong as a statement of fact.

The warning remains relevant as a risk notice. However, it is not a published Midea roadmap. Above all, new technical material is now available that calls an essential part of the earlier interpretation into question.

## How local PortaSplit control works

The Home Assistant integration `Midea Smart AC` explicitly describes its architecture as local control. On newer V3 devices, the Midea cloud is used only during setup to obtain a device-specific token and key. The integration then stores both values locally and needs no further cloud connection for the actual control. The project documents this under [“Note On Cloud Usage”](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

In simplified terms, the process looks like this:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

For manually configured V3 devices, `Midea Smart AC` requires the device ID, IP address, port, token, and key. The documented default port is `6444/TCP`; the token and key are specified as 128 and 64 hexadecimal characters, respectively. These details are provided in the [manual configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

For example, a PortaSplit was identified in the issue tracker of `Midea AC LAN` as device type `0xAC`, model `00000Q1D`, and protocol version 3. The same user was then able to add it to Home Assistant through NetHome Plus. The specific history is documented in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

The key distinction is:

- The cloud service is used to obtain the local credentials.
- Subsequent control takes place directly on the LAN.
- A disruption of the token service therefore primarily prevents new setups.
- It does not automatically end an already configured local connection.

The latter also matches the explicit description from [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Where the shutdown warning comes from

The warning text visible today was added to the documentation on May 19, 2025 through [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

In summary, its rationale is:

- Local tokens have no expiration date.
- Various Home Assistant projects used emulated or extracted app encryption.
- This creates a security issue.
- Midea would therefore gradually close the existing token services.
- In the long term, local V1 control would be displaced by a cloud-based V2 API.

In July 2025, the documentation was revised again through [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Instead of the SmartHome cloud, NetHome Plus was now named as the temporarily used token source. The actual shutdown warning remained unchanged.

However, the underlying discussion is phrased more cautiously than the README.

In the [comment by the Midea AC LAN maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), he essentially says that NetHome Plus may be only a temporary solution and that, to his understanding, Midea has a new fully cloud-based V2 service.

The maintainer of `midea-msmart` replied that he had also suspected the existence of a new V2 API, but could investigate it only to a limited extent because he did not own any Midea devices. This is stated in the [direct reply comment](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

This makes the source situation clearer:

- The warning comes from experienced community developers.
- It is based on observed changes and their technical assessment.
- One maintainer explicitly describes the V2 migration as his understanding.
- The other calls it a suspicion.
- Neither the pull request nor the discussion links to an official Midea shutdown announcement or a date.

That does not make the warning worthless. But it makes it a risk assessment rather than a confirmed vendor roadmap.

## The decisive new finding from June 2026

On June 15, 2026, a fix was merged into library `midea-local` that substantially changes the previous interpretation.

The starting point was the error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

This error occurred while retrieving the token and key through the SmartHome cloud. Login and the device list continued to work, but the call to `/v1/iot/secure/getToken` was rejected.

At first, this looked like an interface that had been shut down or rendered unusable. However, an analysis of the official SmartHome app’s request revealed a different cause: in addition to `udpid`, the app also sent the field `applianceCodes`. The community library had not sent that field.

The corrected request now includes:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

The developer tested the change with a real SmartHome account and four V3 air conditioners of type `0xAC`:

- Without `applianceCodes`, the server returned error 3004.
- With `applianceCodes`, it returned valid tokens and keys.
- The returned values subsequently worked for local V3 authentication.

The full investigation, test results, and code diff are documented in [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). The associated immutable commit is [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

The current source code also continues to use exactly this endpoint:

```text
/v1/iot/secure/getToken
```

In addition, it now sends `applianceCodes`. This can be verified directly in the current [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Since August 12, 2026, `Midea AC LAN` has used its own library `midea-lan` instead of `midea-local`; the current version includes `midea-lan==2026.10.0` and continues to declare itself as a `local_push` integration. Both are stated in the current [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json). `midea-lan` also retrieves tokens and keys from the SmartHome cloud through `/v1/iot/secure/getToken` with `applianceCodes`, as can be seen in [`midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

The blanket statement that the SmartHome token API had been closed is therefore disproven, at least for the accounts and devices tested in June 2026. The accurate statement would be:

> The previous token request stopped working after a change to the expected request format. Once it was adapted to the format used by the official app, the same V1 endpoint again returned valid local credentials.

This does not rule out regional differences, different accounts, or unsupported device types. But it clearly was not a global shutdown.

The situation has since changed for other Midea clouds. Since August 2026, the Chinese Meiju cloud has responded to `/v1/iot/secure/getToken` with error `40404`; it now provides tokens and keys only through the app endpoint `/v2/iot/secure/getToken`, which additionally requires a `homegroupId`. The libraries were adjusted accordingly in late August 2026, as documented in [`midea-local` Pull Request #717](https://github.com/midea-lan/midea-local/pull/717). According to the [README of `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice), tokens can currently no longer be retrieved through NetHome Plus accounts; only personal Meiju and SmartHome accounts to which the device is linked can be used. This change also concerns a private app endpoint, not the official cloud-to-cloud API V2.

## Why “V2” is so easily misunderstood here

At least three independent version designations are used in the Midea ecosystem.

| Term | Meaning |
| --- | --- |
| Local V2/V3 protocol | Generation of direct communication between the integration and device |
| V1/V2 app endpoint | Version number of an individual HTTP endpoint in the backend of Midea apps |
| Cloud-to-cloud API V2 | Official partner API for authorized third-party companies |

### Local V2 and V3

In the local device protocol, V2 and V3 refer to the device’s communication generation. Newer V3 devices require a token and key for local authentication. `Midea Smart AC` documents this requirement in its [configuration guide](https://github.com/mill1000/midea-ac-py#manual-configuration).

This protocol version has nothing to do with the official cloud-to-cloud API V2.

### V1 and V2 in app URLs

Even within the same app, endpoints with different version numbers can be used simultaneously. A `/v2/` in the URL path therefore does not mean the entire platform has been switched to a new architecture.

The current `midea-local` code still uses [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for tokens and keys. Other functions may nevertheless be located under differently versioned paths.

### Official cloud-to-cloud API V2

Midea does in fact document an [official cloud-to-cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

It uses, among other things:

- OAuth 2.0
- `client_id` and `client_secret`
- short-lived access tokens and refresh tokens
- HMAC-SHA256 signatures
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- cloud-based status queries and control commands

This is a controlled partner interface. The required `client_secret` is assigned by Midea to a third-party vendor. A regular PortaSplit owner does not simply receive it through their MSmartHome account. The requirements and signature rules are described in the [official V2 documentation](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

This API did not originate only in 2025, either. The documentation includes request examples with timestamps from 2018 and a Java comment dated April 18, 2019. The V2 partner interface thus existed well before the warning in `Midea AC LAN`.

## Midea is actually replacing a V1 API—but a different one

Midea also maintains an older official cloud-to-cloud interface under `/v1/open/...`. Its documentation explicitly notes that it is no longer recommended, may be shut down in the future, and that the new V2 documentation should be used. This is stated in Midea’s [documentation for the old cloud-to-cloud API](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

This notice is a genuine official V1-to-V2 migration. But it concerns the partner endpoints:

```text
/v1/open/...
           ↓
/v2/open/...
```

By contrast, the token request used by the Home Assistant libraries is:

```text
/v1/iot/secure/getToken
```

And the local PortaSplit connection does not use such a cloud URL at all afterward; it runs directly on the home network.

It would therefore not be technically justified to equate the three interfaces merely because of the version number “V1.”

## Is there already a fully cloud-based Home Assistant integration?

A community integration now exists with [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) that controls Midea devices through the cloud rather than directly over the LAN.

However, this is not evidence that the official partner V2 API has already replaced local control. The current source code of `Midea Auto Cloud` uses, among other things:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

These endpoints can be viewed in the current [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

The integration therefore emulates private app or consumer-cloud functions. It does not simply use the documented `/v2/open/...` partner interface.

A cloud-based alternative already exists. But it also brings the usual dependencies of a cloud integration: internet access, a functioning user account, available Midea servers, and still-compatible private endpoints.

## What does this specifically mean for PortaSplit owners?

### Already configured local control

For an already configured PortaSplit, the situation is relatively uncritical. `Midea Smart AC` stores the token and key locally after setup and, according to its own [cloud documentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), needs no cloud connection for subsequent control.

Shutting down pure token retrieval would therefore not automatically terminate the existing local connection.

### New setup or recovery

The risk is greater with:

- a new Home Assistant installation
- switching to another integration
- a lost or damaged backup
- replacing the Wi-Fi module
- changes to device assignment
- pairing again, if this changes the device credentials

In such cases, the integration must obtain the token and key again, or the user must provide them manually. That `Midea Smart AC` supports manual configuration is described in its [configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Whether a factory reset or pairing again necessarily generates new credentials for every PortaSplit is not officially documented and should therefore not be stated categorically.

### A genuine shutdown of LAN control

For an already configured PortaSplit to no longer accept its locally stored credentials, the behavior of the device or Wi-Fi module would also have to change, for example through new firmware or a changed authentication procedure.

Simply shutting down the cloud endpoint `/v1/iot/secure/getToken` does not automatically remove the credentials already present in the device and Home Assistant. This follows from the separation between one-time cloud retrieval and subsequent LAN control documented by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Such a future device change is technically possible. However, I have not found a specific announcement or shutdown date for the PortaSplit in publicly available Midea documentation.

## What I would still recommend

Despite these qualifying findings, a backup remains advisable.

For V3 devices, `Midea AC LAN` explicitly recommends saving the generated JSON configuration outside HAOS. The current recommendation is stated directly in the [project README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

A backup is sensible protection against cloud changes, integration issues, and personal mistakes, but it is not evidence that a shutdown is imminent. [Part 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) explains how to back up the token, key, and configuration.

## Assessment based on the available evidence

The warning from `Midea AC LAN` should be taken seriously, but put into proper context.

It documents a plausible long-term risk: Midea could regard non-expiring local tokens as a security issue, further restrict the acquisition of such tokens, or tie future devices more closely to the cloud.

What is not established, however, is an officially announced, scheduled shutdown of local PortaSplit control.

The current technical status even shows the opposite of a shutdown already having occurred: in June 2026, the still-used V1 token endpoint returned valid credentials after the request had been adapted to the format of the official SmartHome app. The corresponding fix is now part of the library used by `Midea AC LAN`.

The official Midea cloud-to-cloud API V2 also exists. But it is an older, access-restricted partner interface and not automatically the successor to the local PortaSplit protocol.

The sober conclusion is therefore:

> Create a backup, monitor integrations, and keep cloud dependencies in mind—but do not prematurely write off local PortaSplit control based on an unconfirmed shutdown assumption.

## Change log

- **October 25, 2026:** Added: Since August 2026, Midea AC LAN has used the midea-lan library (currently 2026.10.0), and the Meiju cloud provides tokens and keys only through the /v2/iot/secure/getToken endpoint with homegroupId. Updated the article date to October 25, 2026, and added sources.

## Sources

1.  [Midea AC LAN: current README and shutdown warning](https://github.com/wuwentao/midea_ac_lan#1-important-notice): wording of the warning, backup recommendation, and distinction between older V2 and newer V3 devices.

2.  [Midea AC LAN PR #578 from May 19, 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduction of the warning about the gradual shutdown of token services and the claimed migration to a cloud-based V2 API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): switch of the documented token source to NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussion about the faulty SmartHome token request and the temporary use of NetHome Plus.

5.  [Comment by the Midea AC LAN maintainer on the suspected V2 migration](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): explicitly identifies the statement about the new V2 cloud as his own understanding.

6.  [Reply from the midea-msmart maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): describes the existence of a new V2 API as a suspicion and notes the limited reverse-engineering possibilities.

7.  [midea-local PR #470 from June 15, 2026](https://github.com/midea-lan/midea-local/pull/470): analysis of error 3004, capture of the official app request, addition of `applianceCodes`, and successful testing with four V3 air conditioners.

8.  [Immutable commit for the SmartHome getToken fix](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): exact code diff of the merged fix.

9.  [Current midea-local cloud code](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint still in use `/v1/iot/secure/getToken` and current request field `applianceCodes`.

10.  [Current Midea AC LAN manifest](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): version of `midea-lan` used and classification as a local push integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentation of local control, one-time cloud retrieval for V3 devices, and manual configuration with token and key.

12.  [Midea AC LAN Issue #607 on the PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): specific PortaSplit example with device type `0xAC`, model `00000Q1D`, protocol version 3, and successful setup through NetHome Plus.

13.  [Official Midea cloud-to-cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, client ID, client secret, access and refresh tokens, signing method, and `/v2/open/...` endpoints.

14.  [Official Midea cloud-to-cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): official notice that the old `/v1/open/...` partner interface is no longer recommended and may be shut down in the future.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) and [current cloud code](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): community integration for complete cloud control and the private V1 app endpoints actually used.

16.  [midea-local PR #717](https://github.com/midea-lan/midea-local/pull/717): Meiju cloud switch from `/v1/iot/secure/getToken` (error 40404) to `/v2/iot/secure/getToken` with `homegroupId`.

17.  [Current midea-lan cloud code](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py): library used by Midea AC LAN since August 2026, SmartHome token request through `/v1/iot/secure/getToken` with `applianceCodes`.
