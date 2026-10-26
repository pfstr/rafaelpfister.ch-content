---
title: "Which data center processes Claude requests? Tracing the connection and restricting processing to Switzerland or the EU"
navTitle: "Claude data center"
description: "Which stops a request to Claude passes through, why the trail ends at the Cloudflare edge, and what Anthropic discloses about its data centers. It also compares paths with a fixed processing location (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry), the limits of operating exclusively in Switzerland, and the cost of regional binding."
date: "2026-10-02"
kategorie: "Claude"
timeToRead: "12 min to read"
themen:
  - claude
produkte:
  - "claude"
protokolle:
  - "apis"
  - "tcp"
slug: "which-data-center-processes-claude-tracing-the-connection-and-limiting-processing-to"
translationId: "article-d8b7299589ea96f4"
translationOf: claude-rechenzentrum-schweiz-eu
translationSourceHash: 2d2a3c4b5820e6ec99b710813e209ed3b2e6f6fa750028d120a7d5e4bf4e0c17
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:32:29.018Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/which-data-center-processes-claude-tracing-the-connection-and-limiting-processing-to
---

When using Claude through claude.ai, Claude Code, or the API, you cannot determine which data center processes the request. Only the first segment of the connection, up to the nearest edge node, is visible. For companies that submit personal data or confidential content to a language model, that is insufficient: they must be able to demonstrate to customers, privacy consultants, or auditors where processing takes place.

This article shows how far the connection itself can be traced, what is publicly known about Anthropic’s data centers, and which routes can reliably restrict processing to the EU or the EU plus Switzerland area. Prices and regions are current as of October 25, 2026.

**Switzerland focus:** The revised Federal Act on Data Protection (revFADP) is decisive. Disclosure of personal data abroad is permitted under Art. 16 revFADP if the Federal Council certifies that the destination country provides adequate protection (Annex 1 of the Data Protection Ordinance, DPO) or if appropriate safeguards such as Standard Contractual Clauses are in place. EU and EEA countries are on this list; the United States is included only for companies certified under the Swiss-U.S. Data Privacy Framework. Processing in the EU therefore makes the justification considerably easier, but does not replace the other obligations (data processing agreement, informing data subjects, data security).

**EU note:** Companies based in the EU are instead subject to Art. 44 et seq. GDPR. The European Commission has recognized Switzerland as providing an adequate level of data protection (adequacy decision 2000/518/EC, confirmed in January 2024); processing in Zurich is therefore not a problematic third-country transfer for EU companies.

<aside class="pilot-card">
  <span class="pilot-card__tag">New from adeptio · Seeking pilot companies</span>
  <p class="pilot-card__titel">Use Claude and OpenAI without personal data leaving Switzerland</p>
  <p class="pilot-card__lead">Our intermediary layer sits between your applications in Switzerland and the models from Anthropic and OpenAI, anonymizing every request before it leaves the country. For live operation, we are looking for companies that use the product in their day-to-day work.</p>
  <div class="pilot-card__spalten">
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">Your company</p>
      <ul>
        <li>Personal data leaves Switzerland only in anonymized form</li>
        <li>Easier justification to privacy and audit teams</li>
        <li>Access before general availability and a say in the features</li>
        <li>Direct contact with the developers</li>
      </ul>
    </div>
    <div class="pilot-card__spalte">
      <p class="pilot-card__spalte-titel">adeptio</p>
      <ul>
        <li>Insights from real prompts rather than test data</li>
        <li>Latency and response-quality metrics under real load</li>
        <li>References from the Swiss market</li>
      </ul>
    </div>
  </div>
  <div class="pilot-card__aktionen">
    <a class="pilot-card__cta" href="/kontakt">Apply as a pilot company</a>
    <a class="pilot-card__mehr" href="#anonymisierung-vor-dem-modell-pilotunternehmen-gesucht">Details and legal assessment</a>
  </div>
</aside>

## Which stops a request to Claude passes through

A request to `claude.ai` or `api.anthropic.com` passes through three stages:

| Stage | Who operates it | Visible to you? |
|---|---|---|
| Edge node (TLS termination, abuse protection) | Cloudflare, on behalf of Anthropic | Yes, location can be read from the header |
| Internal network to the backend | Cloudflare and Anthropic | No |
| Inference (the model computes) | Anthropic, using compute from AWS, Google Cloud, and its own data centers | No, only the broad setting `us` or `global` |

The hostnames `claude.ai` and `api.anthropic.com` both point to the address `160.79.104.10`. It belongs to the prefix `160.79.104.0/23`, which is registered to Anthropic (AS399358) and, according to RPKI, validly originates from Anthropic. In global routing, however, the prefix is announced exclusively through Cloudflare (AS13335): Anthropic brings its own IP addresses into Cloudflare’s Anycast network. As a result, the same address is answered by every Cloudflare location worldwide, and the connection reaches the nearest edge node.

## Determining the edge node itself

Cloudflare reveals the serving location on a diagnostic page and in the `cf-ray` header. The code after the Ray ID is an IATA airport code: `ZRH` stands for Zurich, `GVA` for Geneva, `FRA` for Frankfurt, and `MRS` for Marseille.

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace
```

The relevant output lines are `colo=` (edge location) and `loc=` (the country to which Cloudflare assigns your source address). On Windows, PowerShell provides the same result:

```powershell
$trace = Invoke-WebRequest -Uri "https://api.anthropic.com/cdn-cgi/trace" -UseBasicParsing
$trace.Content
```

You can read the header using a HEAD request:

```bash
curl -sI https://api.anthropic.com/ \
  | grep -i -E '^(server|cf-ray):'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-s` | Suppresses curl progress output and error messages |
| `-I` | Sends a HEAD request and outputs only the response headers |
| `grep -i` | Searches without regard to case |
| `-E '^(server\|cf-ray):'` | Extended regular expression: only lines beginning with `server:` or `cf-ray:` |

</details>

From a Swiss network, `ZRH` or `GVA` is generally expected. If the Internet egress is elsewhere, such as in a corporate network with centralized egress abroad, the edge node responds there, for example with `colo=FRA` or `colo=MRS`. The edge location therefore follows your network egress, not the processing location.

## Why traceroute ends at the edge

A traceroute shows the route to the edge node and no further:

```powershell
Test-NetConnection -ComputerName api.anthropic.com -TraceRoute -Hops 20
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-ComputerName api.anthropic.com` | Target host for the measurement |
| `-TraceRoute` | Determines the router hops on the route to the target |
| `-Hops 20` | Maximum number of hops to check |

</details>

On Linux and macOS, the equivalent is `traceroute -n api.anthropic.com` (`-n` suppresses DNS resolution of the hops). In the measurement, the last hops before `160.79.104.10` were in the address range `162.158.0.0/15`, which belongs to Cloudflare. The trail then ends: the connection terminates at the edge node, while forwarding to the backend takes place through a new, internal connection that is not visible externally.

RIPEstat shows who owns the prefix and through which network it is announced:

```bash
curl -s "https://stat.ripe.net/data/prefix-overview/data.json?resource=160.79.104.10"
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=160.79.104.0/23"
```

In the second result, all AS paths end at `13335 399358`: Cloudflare is the sole upstream provider for the Anthropic prefix.

Based on my research, there is no public measurement that traces the route all the way to an inference data center, and because of termination at the edge, this is not possible with network tools either. Measurement services such as llmlatency.dev capture only the response time to the first byte at the edge (as of September 2026: 96 ms median from US Central, 199 ms from Germany). Time to First Token could be compared from multiple locations, but it depends more on the model, prompt length, and load than on distance. It therefore does not support a reliable conclusion about a location.

## What is known about Anthropic’s data centers

Anthropic has not yet operated its own publicly attributed inference data center with an address. Its compute capacity comes from partnerships compiled by Tim Cadenbach in the TCDEV blog:

| Location / partner | Key details | Known role |
|---|---|---|
| Project Rainier, New Carlisle (Indiana, USA), Amazon Web Services | Opened October 2025, around USD 11 billion, approximately 500,000 Trainium2 chips, more than 2.2 GW planned | Anthropic is the primary tenant; mainly training |
| Google Cloud | Agreement in October 2025 for up to 1 million TPUs, more than 1 GW starting in 2026 | Regions not disclosed |
| Own data centers with Fluidstack | Announced November 2025, USD 50 billion, locations in Texas and New York, commissioning starting in 2026 | Still under construction |

Training and inference do not necessarily take place in the same location. Inference can run in any AWS or Google Cloud region, and Anthropic does not disclose which location processes the response for a user in Switzerland. Only what the privacy policy states is certain: the contracting party and controller for customers in the EEA, the United Kingdom, and Switzerland is Anthropic Ireland, Limited in Dublin; data is transferred to servers in the United States or other countries outside the EEA based on Standard Contractual Clauses.

## Control directly through Anthropic: United States or global only

Since generation 4.6 models, the Claude API has included the parameter `inference_geo`. It has exactly two values:

| Value | Effect | Price |
|---|---|---|
| `global` (default) | Inference in any available region | List price |
| `us` | Inference exclusively in the United States | List price × 1.1 |

There is no value for the EU or Switzerland. The workspace storage location (`workspace geo`) can currently only be set to `us`. The response reports which setting was applied in the `usage.inference_geo` field, but not a specific region. For Haiku 4.5 and older models, the API responds to the parameter with a 400 error.

For claude.ai and Claude Code with a subscription (Pro, Max, Team, Enterprise), there is no setting for the processing location. The direct Anthropic offerings therefore cannot restrict processing to either the EU or Switzerland. This is possible only through the cloud providers that operate Claude in their own regions.

## AWS Bedrock: EU plus Switzerland area from Zurich

AWS Bedrock operates Claude on AWS infrastructure. According to AWS, Anthropic has no access to customer prompts, responses, or logs. You select the region through the source region of your API call and through the Inference Profile:

| Endpoint type | Model ID prefix | Processing location |
|---|---|---|
| Global Cross-Region Inference | `global.` | Any commercial AWS region worldwide |
| Geographic Cross-Region Inference | `eu.` | Only regions within the geography |
| In-Region | no prefix | Only the region called |

For Switzerland, the relevant region is `eu-central-2` (Zurich). Opus 5.5, Sonnet 5.5, Haiku 4.5, and, since October 7, 2026, Haiku 5.5 are available there, but only as Global or EU profiles and not In-Region. A profile that processes exclusively in Zurich does not exist. When the EU profile is called from Zurich, AWS distributes requests according to the model card (documented for Opus 5.5, Sonnet 5.5, Haiku 5.5, Haiku 4.5, and Sonnet 4.6) across these regions:

| Region | Location |
|---|---|
| `eu-central-2` | Zurich |
| `eu-central-1` | Frankfurt |
| `eu-north-1` | Stockholm |
| `eu-south-1` | Milan |
| `eu-south-2` | Spain |
| `eu-west-1` | Ireland |
| `eu-west-3` | Paris |

Zurich is only a possible destination when the request originates from Zurich. Processing therefore remains within the EU plus Switzerland area. The list returned by the API for your own source region is authoritative:

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
| `bedrock get-inference-profile` | Reads an Inference Profile definition |
| `--region eu-central-2` | Zurich source region; the destination list depends on the source region |
| `--inference-profile-identifier` | ID of the EU profile for Sonnet 5.5 |
| `--query "models[].modelArn"` | Outputs only model ARNs; the region is included in each ARN |

</details>

By default, data is stored only in the source region. An exception applies to content retained for abuse detection: it is stored in the destination region. Transport between regions is encrypted over the AWS network.

### Demonstrating the processing region for each request

Bedrock is the only one of the described paths that logs the actual processing region for each request. CloudTrail writes it to the `additionalEventData.inferenceRegion` field in the source region:

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
| `jq -r '… \| @tsv'` | Extracts timestamp, action, and processing region as a tab-separated line |

</details>

In addition, a Service Control Policy (SCP) in AWS Organizations can block all regions outside the EU and Switzerland. If a profile destination region is blocked, the request fails instead of falling back to another region. With this combination, the processing location is technically enforced and demonstrable for each request.

### Claude Code through Bedrock in the EU

Claude Code can also be operated through Bedrock. In an `eu-*` region, Claude Code automatically selects the `eu.` prefix:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=eu-central-2
claude
```

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_USE_BEDROCK=1` | Switches Claude Code from the Anthropic API to Bedrock |
| `AWS_REGION=eu-central-2` | Zurich source region; this results in the EU profile |

The prefix can be overridden with `ANTHROPIC_BEDROCK_REGION_PREFIX`. AWS authentication uses the standard mechanisms (profile, SSO, environment variables). Billing is through the AWS account, not a Claude subscription.

## Google Vertex AI: EU only, explicitly excluding Switzerland

On Google Cloud (Vertex AI, now under the name Gemini Enterprise Agent Platform), the situation is less favorable for Switzerland:

| Endpoint | Models (selection) | Processing location |
|---|---|---|
| `global` | all | Any Google Cloud region, without assurance |
| Multi-region `eu` (`aiplatform.eu.rep.googleapis.com`) | Opus 5.5, Sonnet 5.5, Haiku 5.5, Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | EU member states only |
| Region `europe-west1` (Belgium) | Haiku 4.5, Sonnet 4.6, Opus 4.6, and older | According to the model page, Europe multi-region |
| Region `europe-west6` (Zurich) | none | Claude not available |

Google explicitly excludes Switzerland from the EU multi-region endpoint: it covers EU member states only; the United Kingdom and Switzerland are not included. Several Swiss guidance pages cite `europe-west6` as a way to use Claude in Zurich; according to Google’s official location table, no Claude model is available there. Google does not document the actual processing region for each request in a log field. For the global endpoint, Google states that the processing region can neither be controlled nor determined.

Vertex AI is therefore suitable for processing exclusively in the EU, but not for processing in Switzerland.

## Microsoft Foundry: currently no EU option

Microsoft offers Claude in Foundry in two variants. “Hosted on Azure” runs on Azure infrastructure, as Global Standard or Data Zone Standard; the Data Zone is available for Claude only in the United States. “Hosted on Anthropic” runs on Anthropic infrastructure, and Microsoft notes that data may be processed outside Azure and outside the selected region. Anthropic lists Foundry for Europe as “Coming soon.” In Microsoft 365 Copilot and Copilot Studio, Anthropic models are exempt from the EU Data Boundary and are disabled by default in the EU, EFTA (including Switzerland), and the United Kingdom.

## Overview: which path assures which processing location

| Path | Switzerland only | EU only | EU plus Switzerland area | Region demonstrable per request |
|---|---|---|---|---|
| claude.ai, Claude Code (subscription) | no | no | no | no |
| Claude API with `inference_geo` | no | no | no (only `us`) | only `us`/`global` |
| AWS Bedrock, source `eu-central-2`, profile `eu.` | no | no | yes | yes (CloudTrail) |
| AWS Bedrock, source in the EU, profile `eu.` | no | yes | yes | yes (CloudTrail) |
| Google Vertex AI, endpoint `eu` | no | yes | (EU portion only) | no |
| Microsoft Foundry | no | no | no | no |

No provider currently offers a path that guarantees Claude processing exclusively in Switzerland. AWS Bedrock with Zurich as the source region and an EU profile comes closest. If it must additionally be excluded that data leaves the EU, such as due to contractual commitments to EU customers, the EU profile can be called from an EU region such as Frankfurt; Zurich is then no longer a destination.

## Anonymization before the model: seeking pilot companies

In most cases, the processing location matters only because the prompt contains personal data or confidential information. This is where a new product from my company, adeptio gmbh, comes in: it acts as an intermediary layer between applications in Switzerland and the models from Anthropic and OpenAI, anonymizing requests before they leave Switzerland. The model receives only content with no personal connection, regardless of the data center in which it computes.

**Switzerland focus:** Information that can no longer be attributed to a specific or identifiable person is not personal data within the meaning of Art. 5(a) revFADP. The rules on disclosure abroad (Art. 16 et seq. revFADP) do not apply to such content. Whether anonymization reaches this threshold depends on the implementation and the specific data; that is precisely what we want to demonstrate in live operation.

For the rollout, we are seeking companies that use Claude or OpenAI models in production, or are planning to introduce them, and that will use the product as pilot companies in their day-to-day work. Both sides benefit:

| Your company | adeptio |
|---|---|
| Personal data and confidential information leave Switzerland only in anonymized form. Justification to privacy consultants, customers, and auditors becomes easier. | Insights from real prompts and workflows rather than test data: which information needs to be recognized and where anonymization removes too much or too little. |
| The choice among Anthropic, OpenAI, Bedrock, or Vertex AI can again be based on model offerings and price rather than processing location alone. | Latency and response-quality metrics under real load. |
| Access to the product before general availability and influence over which data types and use cases it covers first. | Feedback from privacy and audit teams that guides the product’s evidence. |
| Direct contact with developers instead of ticket support. | References from the Swiss market for general availability. |

<aside class="offer-box">
  <span class="offer-box__tag">Seeking pilot companies</span>
  <p><strong>Do you use Claude or OpenAI with personal data?</strong> Use adeptio’s anonymization layer in live operation and help shape its further development.</p>
  <a class="offer-box__cta" href="/kontakt">Get in touch</a>
</aside>

## What regional binding costs

All three cloud providers and Anthropic itself charge a 10% premium over the global endpoint for binding to a region or geography. For Bedrock, pricing is based on the source region; Zurich, Frankfurt, and Northern Virginia cost the same. Routing between regions does not incur an additional charge.

List prices in USD per 1 million tokens (input / output):

| Model | Anthropic API, `global` | Anthropic API, `us` | Bedrock or Vertex, global | Bedrock `eu.` or Vertex `eu` |
|---|---|---|---|---|
| Opus 5.5 | 4.00 / 20.00 | 4.40 / 22.00 | 4.00 / 20.00 | 4.40 / 22.00 |
| Sonnet 5.5 | 2.00 / 10.00 | 2.20 / 11.00 | 2.00 / 10.00 | 2.20 / 11.00 |
| Haiku 5.5 (prompts up to 100,000 tokens) | 0.10 / 0.50 | 0.11 / 0.55 | 0.10 / 0.50 | 0.11 / 0.55 |
| Haiku 4.5 | 1.00 / 5.00 | not available | 1.00 / 5.00 | 1.10 / 5.50 |
| Sonnet 4.6 | 3.00 / 15.00 | 3.30 / 16.50 | 3.00 / 15.00 | 3.30 / 16.50 |

For Vertex AI, the EU column price applies to Haiku 4.5 and Sonnet 4.6 in the `europe-west1` region. An example calculation for an internal application with 50 million input and 10 million output tokens per month:

| Model | Global | EU-bound | Additional cost per month |
|---|---|---|---|
| Sonnet 5.5 | 50 × 2 + 10 × 10 = 200 USD | 220 USD | 20 USD |
| Opus 5.5 | 50 × 4 + 10 × 20 = 400 USD | 440 USD | 40 USD |

Regional binding therefore costs 10% more than the global endpoint. Compared with Anthropic’s direct list price using `inference_geo: "us"`, an EU profile on Bedrock costs the same. The ancillary costs carry more weight in the decision: an AWS or Google Cloud account with organizational policies, logging, and budget alerts must be set up and operated, while the fixed monthly prices of Claude subscriptions are replaced by usage-only billing. Whether the switch pays off depends on volume; a subscription has a fixed per-person price, but offers no control over the processing location.

## Retention and training

In addition to the location, how long data is stored also matters:

- **Anthropic API:** Prompts and responses are not used for training unless explicit consent is provided. Zero Data Retention (ZDR) is available upon request per organization and can be combined with `inference_geo`. Content flagged by abuse detection remains stored for up to two years even with ZDR. For the Fable and Mythos models, 30-day retention is mandatory.
- **AWS Bedrock:** Content does not go to Anthropic. Retention can be selected per model between `none`, `default` and `aws_review`; Fable requires `aws_review` with storage for up to 30 days for AWS review.
- **Google Vertex AI:** Google uses the data for training or fine-tuning only with prior consent.

## Change log

- **October 25, 2026:** Added: Haiku 5.5 (since October 7, 2026) with availability in Zurich and in Google Cloud’s EU multi-region, along with pricing; the destination regions of the EU profile are now also documented for Opus 5.5 and Sonnet 5.5; prices and regions current as of October 25, 2026. Updated source links to the new Google Cloud documentation addresses.

## Sources

1.  [Claude Platform Docs: Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency): parameter `inference_geo`, values `us` and `global`, workspace geo, supported models, field `usage.inference_geo`.

2.  [Claude Platform Docs: Pricing](https://platform.claude.com/docs/en/about-claude/pricing): Anthropic API list prices and factor 1.1 for `inference_geo: "us"`.

3.  [Claude Platform Docs: API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention): Zero Data Retention, exceptions for flagged content and for Fable/Mythos.

4.  [Anthropic: Privacy Policy](https://www.anthropic.com/legal/privacy): Anthropic Ireland as controller for the EEA, UK, and Switzerland; transfer to the United States based on Standard Contractual Clauses.

5.  [Anthropic: Regional compliance](https://claude.com/regional-compliance): overview of which platform provides data residency in which region; Foundry Europe listed as “Coming soon.”

6.  [Claude Platform Docs: Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock): endpoint types by region, 10% premium for regional endpoints.

7.  [AWS: Claude Sonnet 4.6 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-6.html): EU profile destination regions by source region, including Zurich.

8.  [AWS: Claude Sonnet 5.5 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5-5.html): availability in `eu-central-2` only as Geo and Global profiles, EU profile destination regions from Zurich.

9.  [AWS: Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html): data remains within the geography, storage location, blocking destination regions using SCP.

10.  [AWS: Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html): `additionalEventData.inferenceRegion` field in CloudTrail, pricing by source region.

11.  [AWS: Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html): model vendors have no access to prompts, responses, and logs.

12.  [AWS: Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/): list prices for Global and Geo profiles, identical for Zurich, Frankfurt, and Northern Virginia.

13.  [AWS Alps Blog: Cross-region inference for EU data processing in Switzerland](https://aws.amazon.com/blogs/alps/unlocking-ai-flexibility-in-switzerland-a-guide-to-cross-region-inference-for-eu-data-processing-and-model-access/): AWS Switzerland guide to the EU profile from Zurich.

14.  [Claude Code Docs: Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock): environment variables and automatic `eu.` prefix in EU regions.

15.  [Google Cloud: Generative AI locations](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/locations): location table for Claude models, no availability in `europe-west6`.

16.  [Google Cloud: Data residency](https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency): EU multi-region only for EU member states; Switzerland and the UK excluded.

17.  [Google Cloud: Vertex AI Pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing): list prices for global, EU multi-region, and `europe-west1`.

18.  [Microsoft Learn: Claude models hosting comparison](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models-hosting-comparison): Hosted on Azure and Hosted on Anthropic, Data Zone only in the United States.

19.  [Microsoft Learn: Anthropic as AI subprocessor in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/connect-to-ai-subprocessor): exception from the EU Data Boundary, default setting in the EU/EFTA.

20.  [TCDEV Blog: Where Are Claude's Data Centers?](https://www.tcdev.de/blog/where-are-claudes-data-centers/): Tim Cadenbach’s overview of Project Rainier, the Google TPU agreement, and own data centers with Fluidstack.

21.  [Anthropic: USD 50 billion investment in U.S. infrastructure](https://www.anthropic.com/news/anthropic-invests-50-billion-in-american-ai-infrastructure): announcement of own data centers in Texas and New York with Fluidstack.

22.  [Claude Platform Docs: IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): incoming address range `160.79.104.0/23`.

23.  [RIPEstat](https://stat.ripe.net/): prefix overview and BGP status for `160.79.104.0/23` (AS399358, upstream AS13335).

24.  [llmlatency.dev: Anthropic](https://llmlatency.dev/provider/anthropic): edge response times by measurement location.

25.  [Fedlex: Data Protection Ordinance (DPO), Annex 1](https://www.fedlex.admin.ch/eli/cc/2022/568/de): list of countries with adequate data protection, including the EU/EEA and the United States for certified companies.

26.  [Anthropic: Claude Haiku 5.5](https://www.anthropic.com/news/claude-haiku-5-5): released October 7, 2026, availability through AWS, Google Cloud, and Microsoft, prices with tier beginning at 100,000 tokens.

27.  [AWS: Claude Haiku 5.5 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html): availability in `eu-central-2` as Geo and Global profiles and EU profile destination regions from Zurich.
