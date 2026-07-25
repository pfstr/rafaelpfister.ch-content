---
title: "Confusion Around the Midea v2 Cloud API: An Attempt to Bring Clarity to the API Jungle"
navTitle: "Midea V2 Cloud API"
description: "Midea AC LAN warns that it will shut down its existing token interfaces. A closer look at the sources and the current code paints a much more nuanced picture: the local device protocol, the app endpoint, and the official partner API all share the same version numbers but mean entirely different things."
date: "2026-07-25"
kategorie: "Smart Home & IoT"
timeToRead: "11 min to read"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-integration"
  - "midea-portasplit-home-assistant-setup-and-hardening"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
slug: "midea-v2-cloud-api-clarified-portasplit-home-assistant"
url: "https://rafaelpfister.ch/en/blog/midea-v2-cloud-api-clarified-portasplit-home-assistant"
---

The warning in the `Midea AC LAN` project sounds dramatic: Midea is said to be closing its existing token interfaces, switching to a new cloud-based V2 API, and thereby, in the long run, rendering local control unusable as well. That is still what the [Midea AC LAN README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) says today.

A closer look at how this warning came about, the discussions around it, the current source code, and the official Midea documentation, however, shows a considerably less clear-cut picture.

The short answer:

> An official Midea Cloud-to-Cloud API V2 does exist. But it is not the same as the token interface used by Home Assistant, nor is it the same as the local V2 or V3 device protocol. No officially announced shutdown of local PortaSplit control with a concrete date is documented. In June 2026 it was also demonstrated that the supposedly shut-down SmartHome token API still worked — the community library's previous request was simply incomplete.

This article reflects the state as of July 25, 2026.

## Why This Follow-Up Is Necessary

In the [first article on the cloud token question](/en/blog/midea-portasplit-home-assistant-integration) I reproduced the warning from the `Midea AC LAN` project, in substance, as an announced shutdown of the cloud interfaces. That matched the wording of the project README, but it was phrased too strongly as a statement of fact.

The warning remains relevant as a risk notice. It is not, however, a published Midea roadmap. Above all, new technical material has since become available that calls a substantial part of the previous interpretation into question.

## How Local PortaSplit Control Works

The Home Assistant integration `Midea Smart AC` explicitly describes its architecture as local control. For newer V3 devices, the Midea cloud is used only during setup, to obtain a device-specific token and key. After that, the integration stores both values locally and needs no further cloud connection for actual control. The project documents this under [“Note On Cloud Usage”](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Simplified, the flow looks like this:

```text
Setup:

Home Assistant
    │
    ├── sign-in to a Midea cloud
    ├── retrieval of device ID, token, and key
    └── local storage of the credentials

Normal operation:

Home Assistant
    │
    └── local TCP connection to the PortaSplit
```

For manually configured V3 devices, `Midea Smart AC` requires device ID, IP address, port, token, and key. The documented default port is `6444/TCP`; token and key are given as 128 and 64 hexadecimal characters, respectively. These details are in the [manual configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

A PortaSplit was, for example, identified in the `Midea AC LAN` issue tracker as device type `0xAC`, model `00000Q1D`, and protocol version 3. The same user was then able to add it to Home Assistant via NetHome Plus. The specific course of events is documented in [issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

The key distinction is:

- The cloud service is used to obtain the local credentials.
- Subsequent control happens directly on the LAN.
- A disruption of the token service therefore mainly prevents new setups.
- It does not automatically end an already established local connection.

The latter also matches the explicit description given by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Where the Shutdown Warning Comes From

The warning text visible today was added to the documentation on May 19, 2025, with [pull request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

The reasoning, summarized, is:

- The local tokens have no expiry date.
- Various Home Assistant projects used reconstructed or extracted app encryption.
- This creates a security problem.
- Midea would therefore progressively shut down the existing token services.
- In the long run, local V1 control would be superseded by a cloud-based V2 API.

In July 2025, the documentation was adjusted again via [pull request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). Instead of the SmartHome cloud, NetHome Plus was now named as the temporarily used token source. The actual shutdown warning remained in place.

The underlying discussion, however, is worded more cautiously than the README.

The [comment from the Midea AC LAN maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) says, in substance, that NetHome Plus might only be a temporary solution and that, as he understands it, Midea has a new, fully cloud-based V2 service.

The maintainer of `midea-msmart` replied that he had suspected the existence of a new V2 API as well, but, lacking his own Midea devices, could only investigate it to a limited extent. That is stated in the [direct reply comment](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

That makes the state of the sources clearer:

- The warning comes from experienced community developers.
- It is based on observed changes and their technical assessment.
- One of the maintainers explicitly describes the V2 migration as his own understanding.
- The other calls it a suspicion.
- Neither the pull request nor the discussion links to an official Midea shutdown announcement or a date.

That does not make the warning worthless. But it makes it a risk analysis, not a confirmed manufacturer roadmap.

## The Decisive New Finding From June 2026

On June 15, 2026, a fix was merged into the `midea-local` library that substantially changes the previous interpretation.

The starting point was the error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

This error occurred when querying token and key via the SmartHome cloud. Login and the device list kept working, but the call to `/v1/iot/secure/getToken` was rejected.

At first, this looked like a shut-down or disabled interface. An analysis of the official SmartHome app's request, however, revealed a different cause: the app also sent an `applianceCodes` field alongside `udpid`. The community library had not been sending that field.

The corrected request now includes:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

The developer tested the change with a real SmartHome account and four V3 air conditioners of type `0xAC`:

- Without `applianceCodes`, the server responded with error 3004.
- With `applianceCodes`, it delivered valid tokens and keys.
- The returned values then worked for local V3 authentication.

The complete investigation, the test results, and the code diff are documented in [`midea-local` pull request #470](https://github.com/midea-lan/midea-local/pull/470). The corresponding immutable commit is [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

The current source code, too, still uses exactly this endpoint:

```text
/v1/iot/secure/getToken
```

In addition, `applianceCodes` is now sent along with it. This can be verified directly in the current [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

The current version of `Midea AC LAN` pulls in `midea-local==6.11.0` and still declares itself a `local_push` integration. Both are stated in the current [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

The blanket claim that the SmartHome token API had been shut down is thus refuted, at least for the accounts and devices tested in June 2026. The correct statement would be:

> The previous token query stopped working after a change to the expected request format. After adapting it to the format used by the official app, the same V1 endpoint again returned valid local credentials.

Regional differences, different accounts, or unsupported device types are not ruled out by this. But it evidently was not a global shutdown.

## Why “V2” Is So Easily Misunderstood Here

At least three independent version labels are used in the Midea ecosystem.

| Term | Meaning |
| --- | --- |
| Local V2/V3 protocol | Generation of direct communication between the integration and the device |
| V1/V2 app endpoint | Version number of an individual HTTP endpoint in the backend of the Midea apps |
| Cloud-to-Cloud API V2 | Official partner API for authorized third-party companies |

### Local V2 and V3

For the local device protocol, V2 and V3 denote the device's communication generation. Newer V3 devices require token and key for local authentication. `Midea Smart AC` documents this requirement in its [configuration guide](https://github.com/mill1000/midea-ac-py#manual-configuration).

This protocol version has nothing to do with the official Cloud-to-Cloud API V2.

### V1 and V2 in App URLs

Even within the same app, endpoints with different version numbers can be used at the same time. A `/v2/` in a URL path therefore does not mean that the entire platform has been switched to a new architecture.

The current `midea-local` code still uses [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for token and key. Other functions can nonetheless sit under differently versioned paths.

### Official Cloud-to-Cloud API V2

Midea does in fact document an [official Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Among other things, it uses:

- OAuth 2.0
- `client_id` and `client_secret`
- short-lived access tokens and refresh tokens
- HMAC-SHA256 signatures
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- cloud-based status queries and control commands

This is a controlled partner interface. The required `client_secret` is issued by Midea to a third-party company. An ordinary PortaSplit owner does not simply get one through their MSmartHome account. The requirements and signing rules are described in the [official V2 documentation](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

This API also did not first appear in 2025. The documentation contains request examples with timestamps from 2018 and a Java comment dated April 18, 2019. The V2 partner interface therefore already existed well before the warning in `Midea AC LAN`.

## Midea Really Is Replacing a V1 API — Just a Different One

Midea also runs an older official cloud-to-cloud interface under `/v1/open/...`. Its documentation explicitly notes that it is no longer recommended, may be shut down in the future, and that the new V2 documentation should be used instead. That is stated in Midea's [documentation of the old cloud-to-cloud API](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

This notice is a genuine, official V1-to-V2 migration. But it concerns the partner endpoints:

```text
/v1/open/...
           ↓
/v2/open/...
```

The token query used by the Home Assistant libraries, by contrast, is:

```text
/v1/iot/secure/getToken
```

And the local PortaSplit connection afterward does not run over such a cloud URL at all, but directly on the home network.

Equating the three interfaces just because they share the version label “V1” would therefore not be technically justified.

## Is There Already a Fully Cloud-Based Home Assistant Integration?

With [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud), a community integration now exists that controls Midea devices via the cloud instead of directly over the LAN.

This, too, however, is no evidence that the official partner V2 API has already replaced local control. The current source code of `Midea Auto Cloud` uses, among others:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

These endpoints can be seen in the current [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

The integration thus reimplements private app or consumer-cloud functions. It does not simply use the documented `/v2/open/...` partner interface.

A cloud-based alternative therefore already exists. But it comes with the usual dependencies of a cloud integration: internet access, a working user account, available Midea servers, and continued compatibility of private endpoints.

## What Does This Mean Concretely for PortaSplit Owners?

### Already Set Up Local Control

For an already configured PortaSplit, the situation is relatively relaxed. `Midea Smart AC` stores token and key locally after setup and, according to its own [cloud documentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), needs no cloud connection for further control.

A shutdown of the pure token retrieval would therefore not automatically end the existing local connection.

### New Setup or Recovery

The risk is greater in the case of:

- a new Home Assistant installation
- switching to a different integration
- a lost or damaged backup
- replacing the Wi-Fi module
- changes to the device assignment
- re-pairing, if the device credentials change in the process

In such cases, the integration has to obtain token and key again, or the user has to enter them manually. That `Midea Smart AC` supports manual configuration is described in its [configuration documentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Whether a factory reset or re-pairing necessarily generates new credentials for every PortaSplit is not officially documented and should therefore not be claimed as a blanket rule.

### An Actual Shutdown of LAN Control

For an already set-up PortaSplit to stop accepting its locally stored credentials, the behavior of the device or Wi-Fi module would additionally have to change — for example through new firmware or a changed authentication procedure.

A mere shutdown of the cloud endpoint `/v1/iot/secure/getToken` does not automatically remove the credentials already present on the device and in Home Assistant. This follows from the separation, documented by [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), between a one-time cloud retrieval and subsequent LAN control.

Such a future device change is technically possible. A concrete announcement or shutdown date specifically for the PortaSplit, however, is something I have not found in the publicly available Midea documentation.

## What I'd Still Recommend

Despite these tempering findings, a backup remains sensible.

For V3 devices, `Midea AC LAN` explicitly recommends backing up the generated JSON configuration outside of HAOS. The current recommendation is stated directly in the [project README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

The following applies:

- Treat token and key like passwords.
- Do not upload the JSON file to a public Git repository.
- Do not publish unredacted debug logs.
- Encrypt the backup.
- Additionally create a complete Home Assistant backup.
- Check current functionality before firmware and integration updates.
- Test local control again after updates.

A backup is a reasonable safeguard against cloud changes, integration issues, and your own mistakes. It is not, however, an indication that a shutdown is imminent. How to set up and secure a PortaSplit on the home network properly is covered in the [practical part on setup](/en/blog/midea-portasplit-home-assistant-setup-and-hardening).

## Conclusion

The warning from `Midea AC LAN` should be taken seriously, but put in the correct context.

It documents a plausible long-term risk: Midea could regard non-expiring local tokens as a security problem, further restrict the acquisition of such tokens, or tie future devices more closely to the cloud.

What is not substantiated, on the other hand, is an officially announced, dated shutdown of local PortaSplit control.

The current technical state even shows the opposite of an already completed shutdown: in June 2026, the still-used V1 token endpoint delivered valid credentials once the request had been adapted to the format of the official SmartHome app. The corresponding fix is now part of the library used by `Midea AC LAN`.

The official Midea Cloud-to-Cloud API V2 does exist as well. But it is an older, access-restricted partner interface, not automatically the successor to the local PortaSplit protocol.

The sober conclusion is therefore:

> Make a backup, keep an eye on the integrations, and keep cloud dependencies in mind — but don't write off local PortaSplit control prematurely based on an unconfirmed shutdown assumption.

## Sources

1.  [Midea AC LAN: current README and shutdown warning](https://github.com/wuwentao/midea_ac_lan#1-important-notice) — Wording of the warning, backup recommendation, and the distinction between older V2 and newer V3 devices.

2.  [Midea AC LAN PR #578 from May 19, 2025](https://github.com/wuwentao/midea_ac_lan/pull/578) — Introduction of the warning about the progressive shutdown of the token services and the claimed migration to a cloud-based V2 API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639) — Switch of the documented token source to NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201) — Discussion of the broken SmartHome token query and the temporary use of NetHome Plus.

5.  [Comment from the Midea AC LAN maintainer on the suspected V2 migration](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) — Explicitly labels the statement about the new V2 cloud as his own understanding.

6.  [Reply from the midea-msmart maintainer](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109) — Describes the existence of a new V2 API as a suspicion and notes the limited reverse-engineering options.

7.  [midea-local PR #470 from June 15, 2026](https://github.com/midea-lan/midea-local/pull/470) — Analysis of error 3004, capture of the official app request, addition of `applianceCodes`, and a successful test with four V3 air conditioners.

8.  [Immutable commit of the SmartHome getToken fix](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5) — Exact code diff of the merged fix.

9.  [Current midea-local cloud code](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) — Still-used endpoint `/v1/iot/secure/getToken` and the current `applianceCodes` request field.

10.  [Current Midea AC LAN manifest](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) — Version of `midea-local` used and classification as a local push integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py) — Documentation of local control, the one-time cloud retrieval for V3 devices, and manual configuration with token and key.

12.  [Midea AC LAN Issue #607 on the PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607) — Concrete PortaSplit example with device type `0xAC`, model `00000Q1D`, protocol version 3, and a successful setup via NetHome Plus.

13.  [Official Midea Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) — OAuth2, client ID, client secret, access and refresh tokens, signing scheme, and `/v2/open/...` endpoints.

14.  [Official Midea Cloud-to-Cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html) — Official notice that the old `/v1/open/...` partner interface is no longer recommended and may be shut down in the future.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) and [current cloud code](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) — Community integration for full cloud control and the private V1 app endpoints it actually uses.
