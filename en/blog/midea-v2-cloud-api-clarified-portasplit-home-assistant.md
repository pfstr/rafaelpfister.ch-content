---
title: "Midea V2, V3 and the Cloud API: What They Actually Mean for the PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "The local device protocol, private app endpoints and the official partner API use similar version names. This source analysis separates these layers and puts the shutdown warning into context."
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
url: "https://rafaelpfister.ch/en/blog/midea-v2-cloud-api-clarified-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-07-29T07:22:23.688Z
translationReview: automatic
translationSourceHash: d63bb09234efede35ce75b1f2ef582fc016791b560d7cb6ba826c718ce50c791
---

In the context of the Midea PortaSplit, ‘V2’ refers to several independent things. There is a local V2 device protocol, version numbers in private app endpoints, and an official cloud-to-cloud API V2 for partners. Anyone who equates these layers will inevitably draw incorrect conclusions about local control.

The project `Midea AC LAN` warns in its [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) that the previous token interfaces would be closed and replaced by a cloud-based V2 API. A review of the discussions, current code and official Midea documentation paints a more nuanced picture:

> An official Midea cloud-to-cloud API V2 exists. However, it is neither identical to the token interface used by Home Assistant nor to the local V2 or V3 device protocol. No officially announced shutdown of local PortaSplit control with a specific date is documented. In June 2026, it was also demonstrated that the supposedly shut-down SmartHome token API was still working – the community library’s previous request was simply incomplete.

This article is current as of 25 July 2026.

## Why the previous assessment needs to be corrected

In the [first article on the cloud token issue](/blog/midea-portasplit-home-assistant), I paraphrased the warning from project `Midea AC LAN` as an announced shutdown of the cloud interfaces. This reflected the wording of the project README, but it was too strongly worded as a statement of fact.

The warning remains relevant as a risk indication. However, it is not a published Midea roadmap. Above all, new technical material is now available that calls a substantial part of the previous interpretation into question.

## How local PortaSplit control works

The Home Assistant integration `Midea Smart AC` explicitly describes its architecture as local control. On newer V3 devices, the Midea cloud is used only during setup to obtain a device-specific token and key. The integration then stores both values locally and requires no further cloud connection for actual control. The project documents this under [“Note On Cloud Usage”](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

For manually configured V3 devices, `Midea Smart AC` requires the device ID, IP address, port, token and key. The documented default port is `6444/TCP`; the token and key are specified as 128 and 64 hexadecimal characters respectively. These details are given in the [manual configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

For example, a PortaSplit was detected in the issue tracker of `Midea AC LAN` as device type `0xAC`, model `00000Q1D` and protocol version 3. The same user was then able to add it to Home Assistant through NetHome Plus. The specific history is documented in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

The distinction is crucial:

- The cloud service is used to obtain the local credentials.
- Subsequent control takes place directly over the LAN.
- A disruption to the token service therefore primarily prevents new setups.
- It does not automatically terminate an already configured local connection.

The latter also corresponds to the explicit description by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Where the shutdown warning originates

The warning text visible today was added to the documentation on 19 May 2025 through [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

In summary, the reasoning is as follows:

- Local tokens had no expiry date.
- Various Home Assistant projects used emulated or extracted app encryption.
- This created a security issue.
- Midea would therefore gradually close the existing token services.
- In the long term, local V1 control would be displaced by a cloud-based V2 API.

In July 2025, the documentation was revised again through [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Instead of the SmartHome cloud, NetHome Plus was now named as the temporarily used token source. The actual shutdown warning remained in place.

However, the underlying discussion is phrased more cautiously than the README.

In the [comment by the Midea AC LAN maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457), it is stated in essence that NetHome Plus may only be a temporary solution and that, as he understands it, Midea has a new, fully cloud-based V2 service.

The maintainer of `midea-msmart` replied that he too had suspected the existence of a new V2 API, but could investigate it only to a limited extent because he did not own any Midea devices. This is stated in the [direct reply comment](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

This makes the source situation clearer:

- The warning comes from experienced community developers.
- It is based on observed changes and their technical assessment of them.
- One maintainer explicitly describes the V2 migration as his understanding.
- The other refers to it as a suspicion.
- Neither the pull request nor the discussion links to an official Midea shutdown announcement or date.

This does not make the warning worthless. But it does make it a risk analysis rather than a confirmed manufacturer roadmap.

## The crucial new finding from June 2026

On 15 June 2026, a fix was merged into library `midea-local` that substantially changes the previous interpretation.

The starting point was the error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

This error had occurred when retrieving the token and key via the SmartHome cloud. Login and the device list continued to work, but the call to `/v1/iot/secure/getToken` was rejected.

At first, this looked like an interface that had been shut down or rendered unusable. However, an analysis of the official SmartHome app’s request revealed a different cause: in addition to `udpid`, the app also sent field `applianceCodes`. The community library had not sent this field.

The corrected request now contains:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

The developer tested the change with a real SmartHome account and four V3 air-conditioning units of type `0xAC`:

- Without `applianceCodes`, the server responded with error 3004.
- With `applianceCodes`, it returned valid tokens and keys.
- The returned values subsequently worked for local V3 authentication.

The full investigation, test results and code diff are documented in [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). The associated immutable commit is [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

The current source code still uses exactly this endpoint:

```text
/v1/iot/secure/getToken
```

In addition, `applianceCodes` is now sent as well. This can be verified directly in the current [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

The current version of `Midea AC LAN` includes `midea-local==6.11.0` and continues to declare itself as a `local_push` integration. Both are stated in the current [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

The blanket claim that the SmartHome token API had been closed is therefore disproven, at least for the accounts and devices tested in June 2026. The correct statement would be:

> The previous token retrieval stopped working after a change to the expected request format. Once it was adapted to the format used by the official app, the same V1 endpoint once again returned valid local credentials.

This does not rule out regional differences, different accounts or unsupported device types. But it was clearly not a global shutdown.

## Why ‘V2’ is so easily misunderstood here

At least three independent version designations are used in the Midea ecosystem.

| Term | Meaning |
| --- | --- |
| Local V2/V3 protocol | Generation of direct communication between the integration and device |
| V1/V2 app endpoint | Version number of an individual HTTP endpoint in the Midea apps’ backend |
| Cloud-to-cloud API V2 | Official partner API for authorised third-party companies |

### Local V2 and V3

For the local device protocol, V2 and V3 denote the device’s communication generation. Newer V3 devices require a token and key for local authentication. `Midea Smart AC` documents this requirement in its [configuration guide](https://github.com/mill1000/midea-ac-py#manual-configuration).

This protocol version has nothing to do with the official cloud-to-cloud API V2.

### V1 and V2 in app URLs

Even within the same app, endpoints with different version numbers can be used simultaneously. A `/v2/` in the URL path therefore does not mean that the entire platform has been moved to a new architecture.

The current `midea-local` code still uses [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for the token and key. Other functions may nevertheless use differently versioned paths.

### Official cloud-to-cloud API V2

Midea does in fact document an [official cloud-to-cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Among other things, it uses:

- OAuth 2.0
- `client_id` and `client_secret`
- short-lived access tokens and refresh tokens
- HMAC-SHA256 signatures
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- cloud-based status queries and control commands

This is a controlled partner interface. The required `client_secret` is allocated to a third-party provider by Midea. An ordinary PortaSplit owner does not simply obtain it through their MSmartHome account. The requirements and signature rules are described in the [official V2 documentation](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

This API did not originate in 2025 either. The documentation contains request examples with timestamps from 2018 and a Java comment dated 18 April 2019. The V2 partner interface therefore existed well before the warning in `Midea AC LAN`.

## Midea is indeed replacing a V1 API – but a different one

Midea also maintains an older official cloud-to-cloud interface under `/v1/open/...`. Its documentation explicitly states that it is no longer recommended, may be shut down in future, and that the new V2 documentation should be used instead. This is stated in Midea’s [documentation for the old cloud-to-cloud API](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

This notice is a genuine official V1-to-V2 migration. However, it concerns the partner endpoints:

```text
/v1/open/...
           ↓
/v2/open/...
```

By contrast, the token retrieval used by the Home Assistant libraries is:

```text
/v1/iot/secure/getToken
```

The local PortaSplit connection then no longer runs via such a cloud URL at all, but directly on the home network.

Equating the three interfaces solely on the basis of the version number ‘V1’ would therefore not be technically justified.

## Is there already a fully cloud-based Home Assistant integration?

With [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), a community integration now exists that controls Midea devices via the cloud instead of directly over the LAN.

However, this too is not evidence that the official partner V2 API has already replaced local control. The current source code of `Midea Auto Cloud` uses, among other things:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

These endpoints can be viewed in the current [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

The integration therefore emulates private app or consumer-cloud functions. It does not simply use the documented `/v2/open/...` partner interface.

A cloud-based alternative therefore already exists. But it also brings the usual dependencies of a cloud integration: internet access, a working user account, available Midea servers and still-compatible private endpoints.

## What does this mean in practice for PortaSplit owners?

### Already configured local control

For an already configured PortaSplit, the situation is relatively relaxed. `Midea Smart AC` stores the token and key locally after setup and, according to its own [cloud documentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), requires no cloud connection for ongoing control.

A shutdown of token retrieval alone would therefore not automatically terminate the existing local connection.

### New setup or recovery

The risk is greater in the event of:

- a new Home Assistant installation
- switching to another integration
- a lost or corrupted backup
- replacement of the Wi-Fi module
- changes to device assignment
- re-pairing, if this changes the device credentials

In such cases, the integration must obtain the token and key again, or the user must provide them manually. That `Midea Smart AC` supports manual configuration is described in its [configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Whether a factory reset or re-pairing necessarily generates new credentials for every PortaSplit is not officially documented and should therefore not be claimed as a general rule.

### A genuine shutdown of LAN control

For an already configured PortaSplit to stop accepting its locally stored credentials, the behaviour of the device or Wi-Fi module would also have to change – for example through new firmware or a changed authentication mechanism.

Merely shutting down cloud endpoint `/v1/iot/secure/getToken` does not automatically remove the credentials already present in the device and Home Assistant. This follows from the separation between one-time cloud retrieval and subsequent LAN control documented by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Such a future device change is technically possible. However, I have not found a specific announcement or shutdown date for the PortaSplit in publicly accessible Midea documentation.

## What I would still recommend

Despite the qualifying findings, a backup remains sensible.

For V3 devices, `Midea AC LAN` explicitly recommends saving the generated JSON configuration outside HAOS. The current recommendation appears directly in the [project README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

The following applies:

- Treat the token and key like passwords.
- Do not upload the JSON file to a public Git repository.
- Do not publish unredacted debug logs.
- Encrypt the backup.
- Also create a complete Home Assistant backup.
- Check that the current setup works before firmware and integration updates.
- Test local control again after updates.

A backup is a sensible safeguard against cloud changes, integration issues and user errors. However, it is not an indication that a shutdown is imminent. The [practical setup guide](/blog/midea-portasplit-home-assistant-einrichten) explains how to set up a PortaSplit properly and secure it on the home network.

## Assessment based on the available evidence

The warning from `Midea AC LAN` should be taken seriously, but interpreted correctly.

It documents a plausible long-term risk: Midea could regard non-expiring local tokens as a security issue, further restrict how such tokens are obtained, or tie future devices more closely to the cloud.

What is not supported, however, is an officially announced and scheduled shutdown of local PortaSplit control.

The current technical state even shows the opposite of a shutdown that has already taken place: in June 2026, the still-used V1 token endpoint returned valid credentials after the request had been adapted to the format of the official SmartHome app. The corresponding fix is now part of the library used by `Midea AC LAN`.

The official Midea cloud-to-cloud API V2 also exists. However, it is an older, access-restricted partner interface and not automatically the successor to the local PortaSplit protocol.

The sober conclusion is therefore:

> Create a backup, monitor integrations and bear cloud dependencies in mind – but do not write off local PortaSplit control prematurely on the basis of an unconfirmed shutdown assumption.

## Sources

1.  [Midea AC LAN: current README and shutdown warning](https://github.com/wuwentao/midea_ac_lan#1-important-notice): wording of the warning, backup recommendation and distinction between older V2 and newer V3 devices.

2.  [Midea AC LAN PR #578 of 19 May 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introduction of the warning about the gradual shutdown of token services and the claimed migration to a cloud-based V2 API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): change of the documented token source to NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): discussion of the faulty SmartHome token retrieval and the temporary use of NetHome Plus.

5.  [Comment by the Midea AC LAN maintainer on the suspected V2 migration](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): explicitly identifies the statement about the new V2 cloud as his own understanding.

6.  [Reply from the midea-msmart maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): describes the existence of a new V2 API as a suspicion and points out the limited reverse-engineering possibilities.

7.  [midea-local PR #470 of 15 June 2026](https://github.com/midea-lan/midea-local/pull/470): analysis of error 3004, capture of the official app request, addition of `applianceCodes` and successful testing with four V3 air-conditioning units.

8.  [Immutable commit for the SmartHome getToken fix](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): exact code diff of the merged fix.

9.  [Current midea-local cloud code](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint `/v1/iot/secure/getToken` still in use and current request field `applianceCodes`.

10.  [Current Midea AC LAN manifest](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): version of `midea-local` in use and classification as a local push integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentation of local control, one-time cloud retrieval for V3 devices and manual configuration with token and key.

12.  [Midea AC LAN Issue #607 on the PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): specific PortaSplit example with device type `0xAC`, model `00000Q1D`, protocol version 3 and successful setup through NetHome Plus.

13.  [Official Midea cloud-to-cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, client ID, client secret, access and refresh tokens, signing method and `/v2/open/...` endpoints.

14.  [Official Midea cloud-to-cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): official notice that the old `/v1/open/...` partner interface is no longer recommended and may be shut down in future.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) and [current cloud code](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): community integration for complete cloud control and the private V1 app endpoints actually used by it.
