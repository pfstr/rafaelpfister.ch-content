---
title: "Which data center does Claude use for computation? Tracing the connection and restricting processing to Switzerland or the EU"
navTitle: "Claude data center"
description: "Which stops a request to Claude passes through, why the trace ends at the Cloudflare edge, and what Anthropic discloses about its data centers. Also: a comparison of routes with a fixed processing location (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), the limits of Switzerland-only operation, and the cost of regional binding."
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
slug: "hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 82442938b810b052119b19044fb328ab5edcdebb9350352ff47207901949421b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:49:31.466Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/hvilket-datasenter-kjorer-claude-i-spor-forbindelsen-og-begrens-behandlingen-til-sveits-eller-eu
aktualisiert: 2026-10-25
---

When using Claude via claude.ai, Claude Code or the API, you cannot find out which data center processes the request. Only the first part of the connection to the nearest edge node is visible. For companies that submit personal data or confidential content to a language model, this is insufficient: they need to be able to document for customers, data protection advisers or auditors where processing takes place.

This article shows how far the connection itself can be traced, what is publicly known about Anthropic's data centers, and which routes can reliably restrict processing to the EU or the EU plus Switzerland area. Prices and regions are current as of 2 October 2026.

**Switzerland focus:** The revised Data Protection Act (revDPA) is decisive. Disclosure of personal data abroad is permitted under Art. 16 revDPA if the Federal Council attests that the destination country provides adequate protection (Annex 1 of the Data Protection Ordinance, DPO) or if appropriate safeguards such as standard contractual clauses are in place. EU and EEA states are on this list, while the US is included only for companies certified under the Swiss-U.S. Data Privacy Framework. A processing location in the EU therefore makes the justification substantially easier, but does not replace the other obligations (data processing agreement, information for data subjects, data security).

**EU note:** Companies based in the EU are instead subject to Art. 44 et seq. GDPR. The European Commission has recognised Switzerland as providing an adequate level of data protection (adequacy decision 2000/518/EC, confirmed in January 2024); processing in Zurich is therefore not a problematic third-country transfer for EU companies.

<aside class="pilot-card">
  <span class="pilot-card__tag">New from adeptio · Pilot companies wanted</span>
  <p class="pilot-card__titel">Use Claude and OpenAI without personal data leaving Switzerland</p>
  <p class="pilot-card__lead">Our intermediary layer sits between your applications in Switzerland and the models from Anthropic and OpenAI, anonymising every request before it leaves the country. For live operation, we are looking for companies that use the product in their day-to-day work.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Your company</p>
      <ul>
        <li>Personal data leaves Switzerland only in anonymised form</li>
        <li>Simpler justification towards data protection and auditing</li>
        <li>Access before general availability and a say in the features</li>
        <li>Direct contact with the developers</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Insights from real prompts rather than test data</li>
        <li>Metrics on latency and response quality under real load</li>
        <li>References from the Swiss market</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Register as a pilot company</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Details and legal classification</a>
  </div>
</aside>

## Which stops a request to Claude passes through

A request to `claude.ai` or `api.anthropic.com` passes through three stages:

| Stage | Who operates it | Visible to you? |
|---|---|---|
| Edge node (TLS termination, abuse protection) | Cloudflare, on behalf of Anthropic | Yes, location can be read from the header |
| Internal network to the backend | Cloudflare and Anthropic | No |
| Inference (the model performs computation) | Anthropic, using computing capacity from AWS, Google Cloud and its own data centers | No, only the broad indication `us` or `global` |

The hostnames `claude.ai` and `api.anthropic.com` both point to the address `160.79.104.10`. It belongs to the prefix `160.79.104.0/23`, which is registered to Anthropic (AS399358) and, according to RPKI, validly originates from Anthropic. In global routing, however, the prefix is announced exclusively through Cloudflare (AS13335): Anthropic brings its own IP addresses into Cloudflare's Anycast network. The same address is thus answered from every Cloudflare location worldwide, and the connection reaches the nearest edge node.

## Determining the edge node itself

Cloudflare reveals the serving location on a diagnostic page and in the `cf-ray` header. The code after the Ray ID is an IATA airport code: `ZRH` stands for Zurich, `GVA` for Geneva, `FRA` for Frankfurt, and `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

The relevant output lines are `colo=` (edge location) and `loc=` (the country to which Cloudflare assigns your source address). On Windows, PowerShell provides the same information:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

You can read the header with a HEAD request:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-s` | Suppresses curl's progress display and error messages |
| `-I` | Sends a HEAD request and outputs only the response headers |
| `grep -i` | Searches without considering upper/lower case |
| `-E '^(server\|cf-ray):'` | Extended regular expression: only lines beginning with `server:` or `cf-ray:` |

</details>

From a Swiss network, `ZRH` or `GVA` can generally be expected. If the internet exit is elsewhere, for example with a corporate network that has a central exit abroad, the edge node responds there, for example with `colo=FRA` or `colo=MRS`. The edge location therefore follows your network exit, not the processing location.

## Why traceroute ends at the edge

A traceroute shows the route to the edge node and no further:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ComputerName api.anthropic.com` | Target host of the measurement |
| `-TraceRoute` | Determines the router hops on the route to the target |
| `-Hops 20` | Maximum number of hops to check |

</details>

On Linux and macOS, this corresponds to `traceroute -n api.anthropic.com` (`-n` suppresses DNS resolution of the hops). In the measurement, the last hops before `160.79.104.10` were in the address range `162.158.0.0/15`, which belongs to Cloudflare. The trace then ends: the connection is terminated at the edge node, while forwarding to the backend runs through a new internal connection that is not visible from outside.

RIPEstat shows who owns the prefix and through which network it is announced:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

In the second result, all AS paths end at `13335 399358`: Cloudflare is the sole upstream of the Anthropic prefix.

According to my research, there is no public measurement that traces the route to an inference data center, and because of termination at the edge, this is also not possible with network tools. Measurement services such as llmlatency.dev record only the response time to the first byte at the edge (as of September 2026: median 96 ms from US Central, 199 ms from Germany). Although time to first token could be compared from several locations, it depends more on the model, prompt length and load than on distance. It therefore does not provide a reliable inference about a location.

## What is known about Anthropic's data centers

Anthropic has not yet operated its own publicly assigned inference data center with an address. Its computing capacity comes from partnerships compiled by Tim Cadenbach in the TCDEV blog:

| Location / partner | Key facts | Known role |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Opened October 2025, around USD 11 billion, approximately 500,000 Trainium2 chips, over 2.2 GW planned | Anthropic is the main tenant; primarily training |
| Google Cloud | Agreement in October 2025 for up to 1 million TPUs, over 1 GW from 2026 | Regions not disclosed |
| Own data centers with Fluidstack | Announced November 2025, USD 50 billion, sites in Texas and New York, commissioning from 2026 | Still under construction |

Training and inference do not necessarily take place in the same location. Inference can run in any AWS or Google Cloud region, and Anthropic does not disclose which location computes the response for a user in Switzerland. The only certainty is what the privacy policy states: Anthropic Ireland, Limited in Dublin is the contracting party and controller for customers from the EEA, the United Kingdom and Switzerland; data is transferred to servers in the US or other countries outside the EEA based on standard contractual clauses.

## Control directly with Anthropic: US or global only

Since generation 4.6 models, the Claude API has supported the `inference_geo` parameter. It has exactly two values:

| Value | Effect | Price |
|---|---|---|
| `global` (default) | Inference in any available region | List price |
| `us` | Inference exclusively in the US | List price × 1.1 |

There is no value for the EU or Switzerland. The Workspace storage location (`workspace geo`) can currently only be set to `us`. The response reports in the `usage.inference_geo` field which setting was applied, but not a specific region. For Haiku 4.5 and older models, the API returns a 400 error for the parameter.

For claude.ai and Claude Code with a subscription (Pro, Max, Team, Enterprise), there is no setting for the processing location. Through Anthropic's direct offerings, processing can therefore be restricted neither to the EU nor to Switzerland. This is only possible through cloud providers that operate Claude in their own regions.

## AWS Bedrock: EU plus Switzerland area from Zurich

AWS Bedrock operates Claude on AWS infrastructure. According to AWS, Anthropic has no access to customers' prompts, responses or logs. You select the region through the source region of your API call and through the Inference Profile:

| Endpoint type | Model ID prefix | Processing location |
|---|---|---|
| Global Cross-Region Inference | `global.` | Any commercial AWS region worldwide |
| Geographic Cross-Region Inference | `eu.` | Only regions within the geography |
| In-Region | without prefix | Only the called region |

For Switzerland, the `eu-central-2` (Zurich) region is decisive. Opus 5.5, Sonnet 5.5 and Haiku 4.5 are available there, but only as Global or EU profiles and not In-Region. A profile that computes exclusively in Zurich does not exist. If the EU profile is called from Zurich, AWS distributes requests according to the model card (documented for Haiku 4.5 and Sonnet 4.6) across these regions:

| Region | Location |
|---|---|
| `eu-central-2` | Zurich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milan |
| `eu-south-2` | Spain |
| `eu-west-1` | Ireland |
| `eu-west-3` | Paris |

Zurich is only a possible destination if the request originates from Zurich. Processing thus remains within the EU plus Switzerland area. For Opus 5.5 and Sonnet 5.5, the model card does not name the target regions; the API returns the actual list:

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
| `--region eu-central-2` | Source region Zurich; the target list depends on the source region |
| `--inference-profile-identifier` | ID of the EU profile for Sonnet 5.5 |
| `--query "models[].modelArn"` | Outputs only the model ARNs; the region appears in each ARN |

</details>

By default, data is stored only in the source region. An exception applies to content retained for abuse detection: it is stored in the target region. Transport between regions is encrypted over the AWS network.

### Documenting the processing region per request

Bedrock is the only one of the described routes that logs the actual processing region for each request. CloudTrail writes it in the source region to the `additionalEventData.inferenceRegion` field:

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
| `--region eu-central-2` | Source region in which the calls are logged |
| `--lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com` | Filters for Bedrock events |
| `--max-results 20` | Limits output to 20 events |
| `--query "Events[].CloudTrailEvent"` | Outputs only the complete event as JSON text |
| `--output text` | Output without JSON wrapper, one event per line |
| `jq -r '… \| @tsv'` | Extracts timestamp, action and processing region as a tab-separated line |

</details>

In addition, a Service Control Policy (SCP) in AWS Organizations can block all regions outside the EU and Switzerland. If a profile's target region is blocked, the request fails instead of falling back to another region. With this combination, the processing location is technically enforced and demonstrable for each request.

### Claude Code via Bedrock in the EU

Claude Code can also be operated through Bedrock. In an `eu-*` region, Claude Code automatically selects the `eu.` prefix:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Switches Claude Code from the Anthropic API to Bedrock |
| `AWS_REGION=eu-central-2` | Source region Zurich; this results in the EU profile |

The prefix can be overridden with `ANTHROPIC_BEDROCK_REGION_PREFIX`. AWS authentication is handled via the usual mechanisms (profile, SSO, environment variables). Billing is through the AWS account, not through a Claude subscription.

## Google Vertex AI: EU only, explicitly excluding Switzerland

On Google Cloud (Vertex AI, now called Gemini Enterprise Agent Platform), the situation is less favourable for Switzerland:

| Endpoint | Models (selection) | Processing location |
|---|---|---|
| `global` | all | Any Google Cloud region, without guarantee |
| Multi-region `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | EU Member States only |
| Region `europe-west1` (Belgium) | Haiku 4.5, Sonnet 4.6, Opus 4.6 and older | According to the model page, Europe multi-region |
| Region `europe-west6` (Zurich) | none | Claude unavailable |

Google explicitly excludes Switzerland from the EU multi-region endpoint: it covers EU Member States only; the United Kingdom and Switzerland are not included. Several Swiss guide pages cite `europe-west6` as a route to Claude in Zurich; according to Google's official location table, no Claude model is available there. Google does not document a log field containing the actual processing region per request. For the global endpoint, Google states that the processing region can neither be controlled nor determined.

Vertex AI is therefore suitable for processing exclusively in the EU, but not for processing in Switzerland.

## Microsoft Foundry: currently no EU option

Microsoft offers Claude in Foundry in two variants. “Hosted on Azure” computes on Azure infrastructure, as Global Standard or Data Zone Standard; the Data Zone is available for Claude only in the US. “Hosted on Anthropic” computes on Anthropic infrastructure, and Microsoft notes that data may be processed outside Azure and outside the selected region. Anthropic lists Foundry for Europe as “Coming soon”. In Microsoft 365 Copilot and Copilot Studio, Anthropic models are exempt from the EU Data Boundary and are disabled by default in the EU, EFTA (including Switzerland) and the United Kingdom.

## Overview: which route guarantees which processing location

| Route | Switzerland only | EU only | EU plus Switzerland area | Region demonstrable per request |
|---|---|---|---|---|
| claude.ai, Claude Code (subscription) | no | no | no | no |
| Claude API with `inference_geo` | no | no | no (only `us`) | only `us`/`global` |
| AWS Bedrock, source `eu-central-2`, profile `eu.` | no | no | yes | yes (CloudTrail) |
| AWS Bedrock, source in the EU, profile `eu.` | no | yes | yes | yes (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | yes | (EU portion only) | no |
| Microsoft Foundry | no | no | no | no |

There is currently no route from any provider that guarantees Claude processing exclusively in Switzerland. AWS Bedrock with Zurich as the source region and an EU profile comes closest. If it must additionally be ruled out that data leaves the EU (for example due to contractual commitments to EU customers), the EU profile can be called from an EU region such as Frankfurt; Zurich then ceases to be a destination.

## Anonymisation before the model: pilot companies wanted

In most cases, the processing location matters only because personal data or confidential information appears in the prompt. This is where a new product from my company, adeptio gmbh, comes in: it acts as an intermediary layer between applications in Switzerland and models from Anthropic and OpenAI, anonymising requests before they leave Switzerland. The model receives only content with no personal reference, regardless of which data center performs the computation.

**Switzerland focus:** Information that can no longer be assigned to a specific or identifiable person is not personal data within the meaning of Art. 5(a) revDPA. The rules on disclosure abroad (Art. 16 et seq. revDPA) do not apply to such content. Whether anonymisation reaches this threshold depends on the implementation and the specific data; this is precisely what we want to demonstrate in live operation.

For the rollout, we are looking for companies that use Claude or models from OpenAI productively, or are planning their introduction, and that use the product as a pilot company in daily work. Both sides benefit:

| Your company | adeptio |
|---|---|
| Personal data and confidential information leave Switzerland only in anonymised form. The justification towards data protection advisers, customers and auditors becomes simpler. | Insights from real prompts and workflows rather than test data: which information must be recognised and where anonymisation removes too much or too little. |
| The choice between Anthropic, OpenAI, Bedrock or Vertex AI can once again be based on the model offering and price rather than solely on the processing location. | Metrics on latency and response quality under real load. |
| Access to the product before general availability and influence over which data types and use cases it covers first. | Feedback from data protection and auditing, against which the product's evidence can be aligned. |
| Direct contact with the developers instead of ticket support. | References from the Swiss market for general availability. |

<aside class="offer-box">
  <span class="offer-box__tag">Pilot companies wanted</span>
  <p><strong>Do you use Claude or OpenAI with personal data?</strong> Use adeptio's anonymisation layer in live operation and help shape its further development.</p>
  <a class="offer-box__cta" href="/kontakt">Get in touch</a>
</aside>

## What regional binding costs

All three cloud providers and Anthropic itself charge a 10% surcharge compared with the global endpoint for binding to a region or geography. For Bedrock, pricing depends on the source region; Zurich, Frankfurt and North Virginia cost the same. Routing between regions does not incur an additional charge.

List prices in USD per 1 million tokens (input / output):

| Model | Anthropic API, `global` | Anthropic API, `us` | Bedrock or Vertex, global | Bedrock `eu.` or Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 4.5 | 1.00 / 5.00 | unavailable | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

For Vertex AI, the EU-column price applies to Haiku 4.5 and Sonnet 4.6 in the `europe-west1` region. A calculation example for an internal application with 50 million input and 10 million output tokens per month:

| Model | Global | EU-bound | Additional monthly cost |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Regional binding therefore costs 10% more than the global endpoint. Compared with Anthropic's direct list price using `inference_geo: "us"`, an EU profile on Bedrock costs the same. Ancillary costs are likely to weigh more heavily: an AWS or Google Cloud account with organisational policies, logging and budget alerts must be established and operated, and fixed monthly Claude subscription prices are replaced by purely usage-based billing. Whether switching is worthwhile depends on volume; a subscription has a fixed price per person, but offers no control over the processing location.

## Retention and training

In addition to the location, how long data is stored also matters:

- **Anthropic API:** Prompts and responses are not used for training unless explicit consent has been provided. Zero Data Retention (ZDR) is available on request per organisation and can be combined with `inference_geo`. Content flagged by abuse detection remains stored for up to two years even with ZDR. For the Fable and Mythos models, retention for 30 days is mandatory.
- **AWS Bedrock:** Content is not passed to Anthropic. Retention can be selected per model between `none`, `default` and `aws_review`; Fable requires `aws_review` with storage of up to 30 days for review by AWS.
- **Google Vertex AI:** Google uses the data for training or fine-tuning only with prior consent.

## Sources

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): Parameter `inference_geo`, values `us` and `global`, Workspace geo, supported models, field `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Anthropic API list prices and factor 1.1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, exceptions for flagged content and for Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland as controller for the EEA, UK and Switzerland, transfer to the US based on standard contractual clauses.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): Overview of which platform offers data residency in which region; Foundry Europe as “Coming soon”.

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): Endpoint types by region, 10% surcharge for regional endpoints.

7.  [AWS: Claude Sonnet 4.6 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): Target regions of the EU profile by source region, including Zurich.

8.  [AWS: Claude Sonnet 5.5 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): Availability in `eu-central-2` only as Geo and Global profile.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): Data remains within the geography, storage location, blocking target regions using SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): `additionalEventData.inferenceRegion` field in CloudTrail, pricing by source region.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): Model providers have no access to prompts, responses and logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): List prices for Global and Geo profiles, identical for Zurich, Frankfurt and North Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): Instructions from AWS Switzerland for the EU profile from Zurich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): Environment variables and automatic `eu.` prefix in EU regions.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/locations): Location table for Claude models, no availability in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency): EU multi-region only for EU Member States, excluding Switzerland and the UK.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing): Global, EU multi-region and `europe-west1` list prices.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure and Hosted on Anthropic, Data Zone only in the US.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): Exemption from the EU Data Boundary, default setting in the EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbach's overview of Project Rainier, the Google TPU agreement and own data centers with Fluidstack.

21.  [Anthropic: USD 50 billion investment in US infrastructure](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): Announcement of own data centers in Texas and New York with Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): Incoming address range `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): Prefix overview and BGP status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): Response times at the edge by measurement location.

25.  [Fedlex: Data Protection Ordinance (DPO), Annex 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): List of states with adequate data protection, including the EU/EEA and the US for certified companies.
