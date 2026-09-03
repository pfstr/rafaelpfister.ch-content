---
title: "Midea V2, V3, and Cloud API: What It Actually Means for the PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "Local device protocols, private app endpoints, and the official partner API use similar version names. Source analysis separates these layers and puts the shutdown warning into context."
date: "2026-07-25"
kategorie: "Home Assistant and IoT"
timeToRead: "11 min read"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-portasplit-home-assistant-einrichten
draft: false
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
slug: "midea-v2-cloud-api-clarified-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:32:32.716Z
translationReview: automatic
translationSourceHash: 12ce029c1de367a718159f3729a8d063f8c7df3982e1a0efa10be83a2af3b3ff
url: https://rafaelpfister.ch/en/blog/midea-v2-cloud-api-clarified-portasplit-home-assistant
---

In the context of the Midea PortaSplit, “V2” refers to several independent things. There is a local V2 device protocol, version numbers in private app endpoints, and an official cloud-to-cloud API V2 for partners. Equating these layers inevitably leads to incorrect conclusions about local control.

The project `Midea AC LAN` warns in its [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) that previous token interfaces would be closed and replaced by a cloud-based V2 API. A review of the discussions, the current code, and the official Midea documentation produces a more nuanced picture:

> An official Midea cloud-to-cloud API V2 exists. However, it is not identical to the token interface used by Home Assistant, nor is it the local V2 or V3 device protocol. An officially announced shutdown of local PortaSplit control with a specific date is not documented. In June 2026, it was also demonstrated that the supposedly discontinued SmartHome token API was still working—the community library's previous request was simply incomplete.

This article is current as of July 25, 2026.

## Why the earlier assessment needs to be corrected

In the [first article on the cloud token question](/blog/midea-portasplit-home-assistant), I had paraphrased the warning from project `Midea AC LAN` as an announced shutdown of the cloud interfaces. This reflected the wording of the project README, but it was too strongly phrased as a statement of fact.

The warning remains relevant as a risk notice. However, it is not a published Midea roadmap. Above all, new technical material is now available that calls a significant part of the previous interpretation into question.

## How local PortaSplit control works

The Home Assistant integration `Midea Smart AC` explicitly describes its architecture as local control. For newer V3 devices, the Midea cloud is used only during setup to obtain a device-specific token and key. The integration then stores both values locally and requires no further cloud connection for actual control. The project documents this under [“Note On Cloud Usage”](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

In simplified form, the process looks like this:

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

For manually configured V3 devices, `Midea Smart AC` requires the device ID, IP address, port, token, and key. The documented default port is `6444/TCP`; the token and key are specified as 128 and 64 hexadecimal characters, respectively. This information is provided in the [manual configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

For example, a PortaSplit was identified in the issue tracker of `Midea AC LAN` as device type `0xAC`, model `00000Q1D`, and protocol version 3. The same user was then able to add it to Home Assistant through NetHome Plus. The specific history is documented in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

The separation is crucial:

- The cloud service is used to obtain the local credentials.
- Subsequent control takes place directly on the LAN.
- A failure of the token service therefore primarily prevents new setups.
- It does not automatically terminate an already configured local connection.

The latter also corresponds to the explicit description by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Where the shutdown warning comes from

The warning text visible today was added to the documentation on May 19, 2025, with [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

In summary, the reasoning is as follows:

- Local tokens have no expiration date.
- Various Home Assistant projects use emulated or extracted app encryption.
- This creates a security issue.
- Midea would therefore gradually close the previous token services.
- In the long term, local V1 control would be displaced by a cloud-based V2 API.

In July 2025, the documentation was revised again through [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Instead of the SmartHome cloud, NetHome Plus was now named as the temporarily used token source. The actual shutdown warning remained in place.

However, the underlying discussion is more cautiously phrased than the README.

In the [comment by the Midea AC LAN maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), he essentially says that NetHome Plus may only be a temporary solution and that, as he understands it, Midea has a new, fully cloud-based V2 service.

The maintainer of `midea-msmart` replied that he had also suspected the existence of a new V2 API, but could investigate it only to a limited extent due to lacking Midea devices of his own. This is stated in the [direct reply comment](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

This makes the source situation clearer:

- The warning comes from experienced community developers.
- It is based on observed changes and their technical assessment.
- One maintainer explicitly characterizes the V2 migration as his understanding.
- The other speaks of a suspicion.
- Neither the pull request nor the discussion links to an official Midea shutdown announcement or date.

That does not make the warning worthless. But it makes it a risk analysis rather than a confirmed manufacturer roadmap.

## The decisive new finding from June 2026

On June 15, 2026, a fix was merged into library `midea-local` that significantly changes the previous interpretation.

The starting point was the error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

This error had occurred when retrieving the token and key through the SmartHome cloud. Login and the device list continued to work, but the call to `/v1/iot/secure/getToken` was rejected.

At first, this looked like an interface that had been shut down or rendered unusable. However, an analysis of the official SmartHome app's request revealed a different cause: In addition to `udpid`, the app also sent field `applianceCodes`. The community library had not sent this field.

The corrected request now includes:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

The developer tested the change with a real SmartHome account and four V3 air conditioners of type `0xAC`:

- Without `applianceCodes`, the server responded with error 3004.
- With `applianceCodes`, it returned valid tokens and keys.
- The returned values subsequently worked for local V3 authentication.

The complete investigation, test results, and code diff are documented in [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). The associated immutable commit is [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

The current source code also continues to use exactly this endpoint:

```text
/v1/iot/secure/getToken
```

In addition, `applianceCodes` is now sent as well. This can be verified directly in the current [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

The current version of `Midea AC LAN` includes `midea-local==6.11.0` and continues to declare itself as a `local_push` integration. Both are stated in the current [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

The blanket claim that the SmartHome token API had been closed is therefore refuted, at least for the accounts and devices tested in June 2026. The correct statement would be:

> The previous token request stopped working after a change to the expected request format. After adapting it to the format used by the official SmartHome app, the same V1 endpoint once again returned valid local credentials.

Regional differences, differing accounts, or unsupported device types are not ruled out. But it was clearly not a global shutdown.

## Why “V2” is so easy to misunderstand here

At least three independent version designations are used in the Midea ecosystem.

| Term | Meaning |
| --- | --- |
| Local V2/V3 protocol | Generation of direct communication between the integration and device |
| V1/V2 app endpoint | Version number of an individual HTTP endpoint in the backend of Midea apps |
| Cloud-to-cloud API V2 | Official partner API for authorized third-party companies |

### Local V2 and V3

For the local device protocol, V2 and V3 refer to the device's communication generation. Newer V3 devices require a token and key for local authentication. `Midea Smart AC` documents this requirement in its [configuration guide](https://github.com/mill1000/midea-ac-py#manual-configuration).

This protocol version has nothing to do with the official cloud-to-cloud API V2.

### V1 and V2 in app URLs

Even within the same app, endpoints with different version numbers can be used simultaneously. A `/v2/` in the URL path therefore does not mean the entire platform has been migrated to a new architecture.

The current `midea-local` code continues to use [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for the token and key. Other functions can nevertheless be located under differently versioned paths.

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

This is a controlled partner interface. The required `client_secret` is assigned by Midea to a third-party provider. A regular PortaSplit owner does not simply obtain it through their MSmartHome account. The requirements and signature rules are described in the [official V2 documentation](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

This API also did not originate in 2025. The documentation contains request examples with timestamps from 2018 and a Java comment dated April 18, 2019. The V2 partner interface therefore existed long before the warning in `Midea AC LAN`.

## Midea is actually replacing a V1 API—but a different one

Midea also maintains an older official cloud-to-cloud interface under `/v1/open/...`. Its documentation explicitly states that it is no longer recommended, may be shut down in the future, and that the new V2 documentation should be used. This is stated in Midea's [documentation for the old cloud-to-cloud API](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

This notice is a genuine official V1-to-V2 migration. However, it concerns the partner endpoints:

```text
/v1/open/...
           ↓
/v2/open/...
```

By contrast, the token request used by the Home Assistant libraries is:

```text
/v1/iot/secure/getToken
```

And the local PortaSplit connection subsequently no longer runs through such a cloud URL at all, but directly on the home network.

It would therefore not be technically justified to equate the three interfaces solely on the basis of the version number “V1.”

## Is there already a fully cloud-based Home Assistant integration?

With [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), there is now a community integration that controls Midea devices through the cloud instead of directly over the LAN.

However, this too is not evidence that the official partner V2 API has already replaced local control. The current source code of `Midea Auto Cloud` uses, among other things:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

These endpoints can be viewed in the current [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

The integration thus emulates private app or consumer cloud functions. It does not simply use the documented `/v2/open/...` partner interface.

A cloud-based alternative therefore already exists. But it also brings the usual dependencies of a cloud integration: internet access, a functioning user account, available Midea servers, and still-compatible private endpoints.

## What does this mean specifically for PortaSplit owners?

### Already configured local control

For an already configured PortaSplit, the situation is comparatively unproblematic. `Midea Smart AC` stores the token and key locally after setup and, according to its own [cloud documentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), does not require a cloud connection for further control.

A shutdown of token retrieval alone would therefore not automatically terminate the existing local connection.

### New setup or recovery

The risk is greater with:

- a new Home Assistant installation
- switching to another integration
- a lost or damaged backup
- replacing the Wi-Fi module
- changes to device assignment
- pairing again, if this changes the device credentials

In such cases, the integration must retrieve the token and key again, or the user must provide them manually. That `Midea Smart AC` supports manual configuration is described in its [configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Whether a factory reset or re-pairing necessarily creates new credentials for every PortaSplit is not officially documented and should therefore not be claimed categorically.

### A real shutdown of LAN control

For an already configured PortaSplit to stop accepting its locally stored credentials, the behavior of the device or Wi-Fi module would also have to change, for example through new firmware or a changed authentication method.

Merely shutting down cloud endpoint `/v1/iot/secure/getToken` does not automatically remove the credentials already present in the device and in Home Assistant. This follows from the separation between one-time cloud retrieval and subsequent LAN control documented by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Such a future device change is technically possible. However, I have not found a specific announcement or shutdown date for the PortaSplit in publicly available Midea documentation.

## What I would still recommend

Despite these qualifying findings, a backup remains sensible.

For V3 devices, `Midea AC LAN` explicitly recommends storing the generated JSON configuration outside HAOS. The current recommendation appears directly in the [project README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

The following applies:

- Treat the token and key like passwords.
- Do not upload the JSON file to a public Git repository.
- Do not publish unredacted debug logs.
- Encrypt the backup.
- Also create a complete Home Assistant backup.
- Check current functionality before firmware and integration updates.
- Test local control again after updates.

A backup is a reasonable safeguard against cloud changes, integration issues, and personal mistakes. However, it is not an indication that a shutdown is imminent. The [practical setup guide](/blog/midea-portasplit-home-assistant-einrichten) explains how to properly set up and secure a PortaSplit on the home network.

## Assessment based on the available evidence

The warning from `Midea AC LAN` should be taken seriously, but placed in the correct context.

It documents a plausible long-term risk: Midea could regard non-expiring local tokens as a security issue, further restrict how such tokens are obtained, or tie future devices more closely to the cloud.

What is not established, however, is an officially announced, scheduled shutdown of local PortaSplit control.

The current technical state even shows the opposite of a shutdown already having taken place: In June 2026, the still-used V1 token endpoint returned valid credentials after the request had been adapted to the format of the official SmartHome app. The corresponding fix is now part of the library used by `Midea AC LAN`.

The official Midea cloud-to-cloud API V2 also exists. But it is an older, access-restricted partner interface and not automatically the successor to the local PortaSplit protocol.

The sober conclusion is therefore:

> Create a backup, monitor integrations, and keep cloud dependencies in mind—but do not prematurely write off local PortaSplit control based on an unconfirmed shutdown assumption.

## Sources

1.  [Midea AC LAN: current README and shutdown warning](https://github.com/wuwentao/midea_ac_lan#1-important-notice): wording of the warning, backup recommendation, and distinction between older V2 and newer V3 devices.

2.  [Midea AC LAN PR #578 of May 19, 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduction of the warning about the gradual shutdown of token services and the claimed migration to a cloud-based V2 API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): change of the documented token source to NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussion of the failed SmartHome token request and temporary use of NetHome Plus.

5.  [Comment by the Midea AC LAN maintainer on the suspected V2 migration](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): explicitly identifies the statement about the new V2 cloud as his own understanding.

6.  [Response from the midea-msmart maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): describes the existence of a new V2 API as a suspicion and points out the limited reverse-engineering options.

7.  [midea-local PR #470 of June 15, 2026](https://github.com/midea-lan/midea-local/pull/470): analysis of error 3004, capture of the official app request, addition of `applianceCodes`, and successful testing with four V3 air conditioners.

8.  [Immutable commit for the SmartHome getToken fix](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): exact code diff of the merged fix.

9.  [Current midea-local cloud code](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): still-used endpoint `/v1/iot/secure/getToken` and current request field `applianceCodes`.

10.  [Current Midea AC LAN manifest](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): version of `midea-local` used and classification as a local push integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentation of local control, one-time cloud retrieval for V3 devices, and manual configuration with token and key.

12.  [Midea AC LAN Issue #607 on the PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): specific PortaSplit example with device type `0xAC`, model `00000Q1D`, protocol version 3, and successful setup through NetHome Plus.

13.  [Official Midea cloud-to-cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, client ID, client secret, access and refresh tokens, signing method, and `/v2/open/...` endpoints.

14.  [Official Midea cloud-to-cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): official notice that the old `/v1/open/...` partner interface is no longer recommended and may be shut down in the future.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) and [current cloud code](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): community integration for fully cloud-based control and the private V1 app endpoints it actually uses.
