---
title: "Which data centre processes Claude? Trace the connection and restrict processing to Switzerland or the EU"
navTitle: "Claude data centre"
description: "Which stages a request to Claude passes through, why the trace ends at the Cloudflare edge and what Anthropic discloses about its data centres. Also compares routes with a fixed processing location (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), the limits of a Switzerland-only operation and the cost of regional binding."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min read"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "which-data-centre-processes-claude-trace-the-connection-and-restrict-processing-to-switzerland"
translationId: "article-d8b7299589ea96f4"
aiPrompt: |
  Du bist mein Berater für Datenstandorte bei KI-Diensten. Hilf mir Schritt für Schritt zu entscheiden, über welchen Weg (Anthropic API, AWS Bedrock, Google Vertex AI oder Microsoft Foundry) wir Claude nutzen sollen, wenn die Verarbeitung in der Schweiz oder in der EU bleiben muss. Frage mich zuerst nach Anwendungsfall, Datenklassifizierung, benötigtem Modell, erwartetem Token-Volumen pro Monat und bestehendem Cloud-Anbieter. Berechne danach die Monatskosten für den globalen und den regional gebundenen Endpunkt und nenne die konkreten Konfigurationsschritte (Region, Inference Profile, Endpoint, Kontrollmöglichkeiten im Log).
translationOf: claude-rechenzentrum-schweiz-eu
url: https://rafaelpfister.ch/sv/blog/which-data-centre-processes-claude-trace-the-connection-and-restrict-processing-to-switzerland
translationSourceHash: dd2606a3781871ddf851af8ceaa22336d00316661e82ca7089325e078f709dc4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:20:01.568Z
translationReview: required
---

Anyone using Claude via claude.ai, Claude Code or the API cannot determine which data centre processes the request. Only the first segment of the connection, up to the nearest edge node, is visible. For companies that submit personal data or confidential content to a language model, this is insufficient: they need to be able to demonstrate to customers, data protection advisers or auditors where processing takes place.

This article shows how far the connection itself can be traced, what is publicly known about Anthropic's data centres and which routes can reliably restrict processing to the EU or the EU-plus-Switzerland area. Prices and regions are current as of 2 October 2026.

**Switzerland focus:** The revised Federal Act on Data Protection (revFADP) is decisive. Disclosure of personal data abroad is permitted under Art. 16 revFADP if the Federal Council certifies that the destination state provides adequate protection (Annex 1 of the Data Protection Ordinance, DPO) or if suitable safeguards such as standard contractual clauses are in place. EU and EEA states are on this list, while the US is included only for companies certified under the Swiss-U.S. Data Privacy Framework. A processing location in the EU therefore makes the justification considerably easier, but does not replace the other obligations (data processing agreement, informing data subjects, data security).

**EU note:** Companies based in the EU are instead subject to Arts. 44 et seq. GDPR. The European Commission has recognised Switzerland as providing an adequate level of data protection (adequacy decision 2000/518/EC, confirmed in January 2024); processing in Zurich is therefore not a problematic third-country transfer for EU companies.

## Which stages a request to Claude passes through

A request to `claude.ai` or `api.anthropic.com` passes through three stages:

| Stage | Who operates it | Visible to you? |
|---|---|---|
| Edge node (TLS termination, abuse protection) | Cloudflare, on behalf of Anthropic | Yes, location can be read from the header |
| Internal network to the backend | Cloudflare and Anthropic | No |
| Inference (the model computes) | Anthropic, using compute from AWS, Google Cloud and its own data centres | No, only the broad designation `us` or `global` |

The host names `claude.ai` and `api.anthropic.com` both resolve to address `160.79.104.10`. It belongs to prefix `160.79.104.0/23`, which is registered to Anthropic (AS399358) and is validly originated by Anthropic according to RPKI. In global routing, however, the prefix is advertised exclusively via Cloudflare (AS13335): Anthropic brings its own IP addresses into Cloudflare's anycast network. The same address is therefore answered from every Cloudflare location worldwide, and the connection reaches the nearest edge node.

## Determine the edge node itself

Cloudflare reveals the serving location on a diagnostic page and in the `cf-ray` header. The abbreviation after the Ray ID is an IATA airport code: `ZRH` stands for Zurich, `GVA` for Geneva, `FRA` for Frankfurt and `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

The relevant lines in the output are `colo=` (edge location) and `loc=` (the country to which Cloudflare assigns your source address). On Windows, PowerShell provides the same result:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

Read the header with a HEAD request:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-s` | Suppresses curl's progress meter and error messages |
| `-I` | Sends a HEAD request and outputs only the response headers |
| `grep -i` | Searches without regard to upper or lower case |
| `-E '^(server\|cf-ray):'` | Extended regular expression: only lines beginning with `server:` or `cf-ray:` |

</details>

From a Swiss network, `ZRH` or `GVA` can generally be expected. If the internet egress is elsewhere, for example in a corporate network with central egress abroad, the edge node there responds, such as with `colo=FRA` or `colo=MRS`. The edge location thus follows your network egress, not the processing location.

## Why traceroute ends at the edge

A traceroute shows the route as far as the edge node, and no further:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ComputerName api.anthropic.com` | Measurement target host |
| `-TraceRoute` | Determines the router hops on the route to the destination |
| `-Hops 20` | Maximum number of hops to check |

</details>

On Linux and macOS, the equivalent is `traceroute -n api.anthropic.com` (`-n` suppresses DNS resolution of the hops). In the measurement, the final hops before `160.79.104.10` were in address range `162.158.0.0/15`, which belongs to Cloudflare. The trace then ends: the connection is terminated at the edge node, while forwarding to the backend occurs over a new internal connection that is not visible externally.

RIPEstat shows who owns the prefix and through which network it is advertised:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

In the second result, all AS paths end at `13335 399358`: Cloudflare is the sole upstream for Anthropic's prefix.

According to my research, no public measurement traces the route through to an inference data centre, and because of termination at the edge, this is not possible with network tools either. Measurement services such as llmlatency.dev capture only the time to first byte at the edge (as of September 2026: median 96 ms from US Central, 199 ms from Germany). While time to first token could be compared across several locations, it depends more on the model, prompt length and load than on distance. It therefore cannot reliably indicate a location.

## What is known about Anthropic's data centres

Anthropic does not yet operate a dedicated inference data centre with a publicly assigned address. The compute capacity comes from partnerships compiled by Tim Cadenbach in the TCDEV blog:

| Location / partner | Key facts | Known role |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Opened October 2025, around USD 11 billion, approximately 500,000 Trainium2 chips, more than 2.2 GW planned | Anthropic is the anchor tenant; primarily training |
| Google Cloud | Agreement in October 2025 for up to 1 million TPUs, more than 1 GW from 2026 | Regions not disclosed |
| Own data centres with Fluidstack | Announced November 2025, USD 50 billion, sites in Texas and New York, commissioning from 2026 | Still under construction |

Training and inference do not necessarily take place in the same location. Inference can run in any AWS or Google Cloud region, and Anthropic does not disclose which location computes the response for a user in Switzerland. The only confirmed information is in the privacy policy: Anthropic Ireland, Limited in Dublin is the contracting party and controller for customers in the EEA, the United Kingdom and Switzerland; data is transferred to servers in the US or other countries outside the EEA, based on standard contractual clauses.

## Control directly at Anthropic: US or global only

Since generation 4.6 models, the Claude API has supported parameter `inference_geo`. It has exactly two values:

| Value | Effect | Price |
|---|---|---|
| `global` (default) | Inference in any available region | List price |
| `us` | Inference exclusively in the US | List price × 1.1 |

There is no value for the EU or Switzerland. The Workspace storage location (`workspace geo`) can currently only be set to `us`. The response reports in field `usage.inference_geo` which setting was applied, but not a specific region. For Haiku 4.5 and older models, the API acknowledges the parameter with a 400 error.

For claude.ai and Claude Code with a subscription (Pro, Max, Team, Enterprise), there is no processing-location setting. Through Anthropic's direct offerings, processing can therefore be restricted neither to the EU nor to Switzerland. This is possible only through cloud providers that operate Claude in their own regions.

## AWS Bedrock: EU-plus-Switzerland area from Zurich

AWS Bedrock runs Claude on AWS infrastructure. According to AWS, Anthropic has no access to customers' prompts, responses or logs. You choose the region through the source region of your API call and through the Inference Profile:

| Endpoint type | Model ID prefix | Processing location |
|---|---|---|
| Global Cross-Region Inference | `global.` | Any commercial AWS region worldwide |
| Geographic Cross-Region Inference | `eu.` | Only regions within the geography |
| In-Region | without prefix | Only the called region |

For Switzerland, region `eu-central-2` (Zurich) is relevant. Opus 5.5, Sonnet 5.5 and Haiku 4.5 are available there, but only as Global or EU profiles and not In-Region. No profile exists that computes exclusively in Zurich. If the EU profile is called from Zurich, AWS distributes requests according to the model card (documented for Haiku 4.5 and Sonnet 4.6) across these regions:

| Region | Location |
|---|---|
| `eu-central-2` | Zurich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milan |
| `eu-south-2` | Spain |
| `eu-west-1` | Ireland |
| `eu-west-3` | Paris |

Zurich is a possible destination only if the request originates from Zurich. Processing therefore remains in the EU-plus-Switzerland area. For Opus 5.5 and Sonnet 5.5, the model card does not state the destination regions; the API returns the actual list:

```bash
aws bedrock get-inference-profile \
  --region eu-central-2 \
  --inference-profile-identifier eu.anthropic.claude-sonnet-5-5 \
  --query "models[].modelArn"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `bedrock get-inference-profile` | Reads the definition of an Inference Profile |
| `--region eu-central-2` | Zurich source region; the destination list depends on the source region |
| `--inference-profile-identifier` | EU profile ID for Sonnet 5.5 |
| `--query "models[].modelArn"` | Outputs only model ARNs; the region is included in each ARN |

</details>

By default, data is stored only in the source region. Content retained for abuse detection is an exception: it is held in the destination region. Transfer between regions is encrypted over the AWS network.

### Document the processing region for each request

Bedrock is the only route described that records the actual processing region for each request. CloudTrail writes it in the source region to field `additionalEventData.inferenceRegion`:

```bash
aws cloudtrail lookup-events \
  --region eu-central-2 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com \
  --max-results 20 \
  --query "Events[].CloudTrailEvent" \
  --output text \
  | jq -r '[.eventTime, .eventName, .additionalEventData.inferenceRegion] | @tsv'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `cloudtrail lookup-events` | Searches CloudTrail events from the past 90 days |
| `--region eu-central-2` | Source region in which calls are logged |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filters for Bedrock events |
| `--max-results 20` | Limits output to 20 events |
| `--query "Events[].CloudTrailEvent"` | Outputs only the complete event as JSON text |
| `--output text` | Output without a JSON wrapper, one event per line |
| `jq -r '… \| @tsv'` | Extracts timestamp, action and processing region as a tab-separated line |

</details>

In addition, a Service Control Policy (SCP) in AWS Organizations can block all regions outside the EU and Switzerland. If a profile's destination region is blocked, the request fails instead of falling back to another region. With this combination, the processing location is technically enforced and verifiable for each request.

### Claude Code via Bedrock in the EU

Claude Code can also run via Bedrock. In an `eu-*` region, Claude Code automatically selects prefix `eu.`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Switches Claude Code from the Anthropic API to Bedrock |
| `AWS_REGION=eu-central-2` | Zurich source region; this results in the EU profile |

Prefix `ANTHROPIC_BEDROCK_REGION_PREFIX` can be used to override the prefix. AWS authentication uses the usual mechanisms (profile, SSO, environment variables). Billing is through the AWS account, not a Claude subscription.

## Google Vertex AI: EU only, explicitly excluding Switzerland

On Google Cloud (Vertex AI, now called Gemini Enterprise Agent Platform), the situation is less favourable for Switzerland:

| Endpoint | Models (selection) | Processing location |
|---|---|---|
| `global` | all | Any Google Cloud region, without assurance |
| Multi-region `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | EU member states only |
| Region `europe-west1` (Belgium) | Haiku 4.5, Sonnet 4.6, Opus 4.6 and older | According to the model page, Europe multi-region |
| Region `europe-west6` (Zurich) | none | Claude unavailable |

Google explicitly excludes Switzerland from the EU multi-region endpoint: it covers EU member states only; the United Kingdom and Switzerland are not included. Several Swiss guidance pages cite `europe-west6` as a route to Claude in Zurich; according to Google's official location table, no Claude model is available there. Google does not document a log field containing the actual processing region for each request. For the global endpoint, Google states that the processing region can neither be controlled nor determined.

Vertex AI is therefore suitable for processing exclusively in the EU, but not for processing in Switzerland.

## Microsoft Foundry: currently no EU option

Microsoft offers Claude in Foundry in two variants. “Hosted on Azure” computes on Azure infrastructure, either as Global Standard or Data Zone Standard; the Data Zone for Claude exists only for the US. “Hosted on Anthropic” computes on Anthropic infrastructure, and Microsoft notes that data may be processed outside Azure and outside the selected region. Anthropic lists Foundry for Europe as “Coming soon”. In Microsoft 365 Copilot and Copilot Studio, Anthropic models are excluded from the EU Data Boundary and are disabled by default in the EU, EFTA (including Switzerland) and the United Kingdom.

## Overview: which route guarantees which processing location

| Route | Switzerland only | EU only | EU-plus-Switzerland area | Region verifiable per request |
|---|---|---|---|---|
| claude.ai, Claude Code (subscription) | no | no | no | no |
| Claude API with `inference_geo` | no | no | no (only `us`) | only `us`/`global` |
| AWS Bedrock, source `eu-central-2`, profile `eu.` | no | no | yes | yes (CloudTrail) |
| AWS Bedrock, source in the EU, profile `eu.` | no | yes | yes | yes (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | yes | (EU portion only) | no |
| Microsoft Foundry | no | no | no | no |

No provider currently offers a route that guarantees Claude processing exclusively in Switzerland. AWS Bedrock with Zurich as the source region and the EU profile comes closest. If it must additionally be ruled out that data leaves the EU (for example, due to contractual commitments to EU customers), the EU profile can be called from an EU region such as Frankfurt; Zurich is then no longer a destination.

## What regional binding costs

All three cloud providers and Anthropic itself charge a 10% premium for binding to a region or geography compared with the global endpoint. For Bedrock, prices are based on the source region; Zurich, Frankfurt and North Virginia cost the same. Routing between regions incurs no additional charge.

List prices in USD per 1 million tokens (input / output):

| Model | Anthropic API, `global` | Anthropic API, `us` | Bedrock or Vertex, global | Bedrock `eu.` or Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | not available | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

For Vertex AI, the EU-column price applies to Haiku 4.5 and Sonnet 4.6 in region `europe-west1`. An example calculation for an internal application with 50 million input and 10 million output tokens per month:

| Model | Global | EU-bound | Additional cost per month |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = USD 200 | USD 220 | USD 20 |
| Opus 5.5 | 50 × 4 + 10 × 20 = USD 400 | USD 440 | USD 40 |

Regional binding therefore costs 10% more than the global endpoint. Compared with Anthropic's direct list price with `inference_geo: "us"`, an EU profile on Bedrock costs the same. Other overheads tend to weigh more heavily in the decision: an AWS or Google Cloud account with organisational policies, logging and budget alerts must be set up and operated, and fixed monthly Claude subscription prices are replaced by pure usage-based billing. Whether switching pays off depends on volume; a subscription has a fixed price per person, but provides no control over the processing location.

## Retention and training

Alongside location, how long data is stored also matters:

- **Anthropic API:** Prompts and responses are not used for training unless explicit consent is given. Zero Data Retention (ZDR) is available on request per organisation and can be combined with `inference_geo`. Content flagged by abuse detection remains stored for up to two years even with ZDR. The Fable and Mythos models require retention for 30 days.
- **AWS Bedrock:** Content is not sent to Anthropic. Retention can be selected per model between `none`, `default` and `aws_review`; Fable requires `aws_review` with storage for up to 30 days for review by AWS.
- **Google Vertex AI:** Google uses the data for training or fine-tuning only with prior consent.

## Källor

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, values `us` and `global`, Workspace geo, supported models, field `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Anthropic API list prices and factor 1.1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, exceptions for flagged content and Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland as controller for the EEA, UK and Switzerland, transfer to the US based on standard contractual clauses.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): overview of which platform provides data residency in which region; Foundry Europe as “Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): endpoint types by region, 10% premium for regional endpoints.

7.  [AWS: Model card Claude Sonnet 4.6](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): destination regions of the EU profile by source region, including Zurich.

8.  [AWS: Model card Claude Sonnet 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): availability in `eu-central-2` only as Geo and Global profiles.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): data remains within the geography, storage location, blocking destination regions using SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): field `additionalEventData.inferenceRegion` in CloudTrail, pricing by source region.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): model providers have no access to prompts, responses and logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): list prices for Global and Geo profiles, identical for Zurich, Frankfurt and North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): guidance from AWS Switzerland on the EU profile from Zurich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): environment variables and automatic prefix `eu.` in EU regions.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): location table for Claude models, no availability in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU multi-region only for EU member states, Switzerland and UK excluded.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): global, EU multi-region and `europe-west1` list prices.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure and Hosted on Anthropic, Data Zone only in the US.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): exclusion from the EU Data Boundary, default setting in EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbach's overview of Project Rainier, the Google TPU agreement and proprietary data centres with Fluidstack.

21.  [Anthropic: USD 50 billion investment in US infrastructure](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): announcement of proprietary data centres in Texas and New York with Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): incoming address range `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): prefix overview and BGP status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): response times at the edge by measurement location.

25.  [Fedlex: Data Protection Ordinance (DPO), Annex 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): list of states with adequate data protection, including EU/EEA and the US for certified companies.
