---
title: "Up to 10 Million Free Tokens per Day: Using OpenAI’s Data-Sharing Program with Cost Guardrails"
navTitle: "OpenAI Free Tokens"
description: "OpenAI gives organizations that share their API traffic for training a daily free allowance: depending on the tier, up to 10 million tokens. With prepaid credit, project limits, and a token budget in the code, usage remains free indefinitely."
date: "2026-08-27"
kategorie: "OpenAI API"
timeToRead: "9 min read"
themen:
  - openai-api
produkte:
  - "openai"
protokolle:
  - "apis"
  - "lizenzierung"
slug: "up-to-10-million-free-tokens-per-day-using-openai-s-data-sharing-program-with-cost-guardrails"
translationId: "article-dde41cbe2dd858e6"
aiPrompt: |
  Du bist mein Assistent für die OpenAI-Plattform. Prüfe mit mir Schritt für Schritt, ob mein OpenAI-Konto für das Data-Sharing-Programm mit Gratis-Tokens sauber abgesichert ist: 1) Billing: Prepaid-Guthaben statt Rechnung, Auto-Reload aus. 2) Data controls → Sharing: "Share inputs and outputs" nur für ein dediziertes Projekt aktiviert, Enrollment-Hinweis sichtbar. 3) Projekt: eigenes Spend-Limit gesetzt, nur ein restricted API-Key. 4) Limits: Spend-Alerts konfiguriert. 5) Code: tägliches Token-Budget deutlich unter Gratis-Kontingent und Tages-Rate-Limit. Frage mich nach meinem Usage-Tier und Modell und rechne mir mein Gratis-Kontingent aus.
translationOf: openai-gratis-tokens-data-sharing
url: https://rafaelpfister.ch/en/blog/up-to-10-million-free-tokens-per-day-using-openai-s-data-sharing-program-with-cost-guardrails
translationSourceHash: 0f0fef78a8ab264b755061045a34cc765916b1f1b433473f99a5eb6e0538a6b0
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:42:39.679Z
translationReview: required
---

# Up to 10 Million Free Tokens per Day: Using OpenAI’s Data-Sharing Program with Cost Guardrails

OpenAI pays for training data with compute rather than money: since December 2024, organizations that share their API inputs and outputs for training have received a daily allowance of free tokens. Depending on the usage tier and model group, this ranges from 250,000 to 10 million tokens per day. For many small automations, this is entirely sufficient: a nightly batch translation, a classification cron job, or automatic tagging of a public archive can remain free indefinitely.

To keep it free, you need limits—and in the right place. A token counter in your own code is a convenience feature; only the limits enforced by OpenAI itself are binding.

## The Program: Tokens in Exchange for Training Data

Participation is managed through the **Share inputs and outputs with OpenAI** setting under *Settings → Data controls → Sharing*. Only the Organization Owner can change it, either for the entire organization or for individual projects. Organizations eligible for the program see the message "You're eligible for free daily usage on traffic shared with OpenAI" on this page; after enabling it, this changes to "You're enrolled for complimentary daily tokens." If the message is absent, the organization is currently ineligible. Accounts with Zero Data Retention and Enterprise agreements are excluded from input-output sharing.

The allowance depends on the organization’s usage tier and is calculated per model group:

| Model group | Tier 1–2 | Tier 3–5 |
|---|---|---|
| Large models (including gpt-5.6-sol, gpt-5.x, o-series, gpt-4.1, gpt-4o) | 250,000 tokens/day | 1 million tokens/day |
| Small models (including gpt-5.6-terra, gpt-5.6-luna, mini and nano variants) | 2.5 million tokens/day | 10 million tokens/day |

The most important rules in detail:

- Input and output tokens are counted together, shared across all models in a group. The counter resets daily at 00:00 UTC.
- Fine-tuned models, fine-tuning training, evals, and tool use are excluded.
- The account needs a positive credit balance; otherwise, even the free tokens will not work.
- OpenAI reserves the right to end the program with 30 days’ notice.

The most important billing rule: the request that exceeds the daily allowance is billed **in full** at the standard rate, not just the excess portion. If you are at 975,000 of 1 million tokens and send a request with 30,000 tokens, you pay for all 30,000. For your own budget planning, this means: build in a safety margin rather than optimizing up to the allowance.

## What You Give Up in Return

The tradeoff is unambiguous: all inputs and outputs from shared projects go to OpenAI and may be used to train future models. This rules out entire categories of use cases. Customer data, support tickets, internal documents, code containing configuration details, and anything containing personal data must not reach a shared project; for Swiss companies, the revised Federal Act on Data Protection (revFADP) already sets the boundary before confidentiality toward customers even becomes an issue.

Workloads involving data that is already public are well suited. One example is the nightly translation of a public blog into several languages: the articles are online, every crawler can already read them today, and the translations are published as well. In such a case, sharing does not disclose anything that has not already been disclosed. Other candidates include alt text for a public image archive, tagging Open Source documentation, or summaries of public release notes for a changelog.

## Setting Up Cost Guardrails in Your OpenAI Account

The order is deliberate: first come the limits that OpenAI enforces server-side. They still apply if your own code contains a bug, a cron job runs twice, or a key falls into the wrong hands.

**Prepaid credit, auto-reload off.** Set billing to "Pay as you go" with prepaid credit and disable automatic reloading. This limits the maximum damage to the remaining credit: once it is depleted, the API rejects further requests. Since the program requires a positive credit balance, you need a small base amount; $5 to $10 is sufficient and remains untouched during normal operation. This step is the only one that truly stops everything in the worst case, which is why it comes first.

**A dedicated project for shared traffic.** Set sharing to "Enabled for selected projects" and share only a project created specifically for this purpose. All other projects in the organization remain excluded from training, and accidental traffic from other applications ends up neither in the training dataset nor in the wrong budget.

**Set a low project spend limit.** Projects have their own monthly spend limit, and it is hard: requests fail as soon as it is reached. For a project that is expected to cost $0, it can be set very low; $5 is enough as a reserve in case a single run exceeds the free allowance. The organization-level limit, by contrast, is intended as a cap with alerts; warning thresholds, such as at 90 and 100 percent, trigger emails.

**One restricted key per project, only as a CI secret.** Create the API key in the project, not at the organization level, and grant it only the permissions the workload needs. For a CI workflow, this means exactly one key with restricted permissions, stored as a secret in the CI environment. It appears in no repository, no local shell, and no second service.

**Choose a model from the inexpensive group.** The difference between the groups is a factor of 10. At Tier 1, a model from the small group gives you 2.5 million tokens per day instead of 250,000. For structured tasks such as translation, classification, or extraction, the small group is generally sufficient.

## The Second Line of Defense in Code

The account limits prevent financial damage, but they lead to hard failures: reaching a spend limit interrupts a run in the middle of a batch. To stay cleanly within the free allowance, you can additionally count tokens yourself. A simple daily counter has proven effective, configured like this, for example:

```json
{
  "openai": {
    "model": "gpt-5.6-terra",
    "reasoningEffort": "none",
    "maxOutputTokens": 32000,
    "dailyTokenBudget": 1000000
  }
}
```

The mechanism behind it consists of four rules:

- After each response, the job adds the `input_tokens` and `output_tokens` reported by the API to a counter in a state file. There is no estimate and no second query—only the usage data from the response itself.
- Before each request, it checks the remaining budget. If there is no longer enough budget to safely accommodate a complete response, the run ends normally with the stop reason `token-budget` rather than with an error.
- The counter uses UTC calendar days and is therefore synchronized with the free allowance reset at 00:00 UTC.
- Independently of the budget, the number of API calls per run is capped so that even a series of failed attempts cannot exhaust the allowance. Transport and quota errors stop the run without automatic retries.

At 1 million, this example’s budget is deliberately well below the allowance of 2.5 million. The margin follows from two characteristics of billing. First, the counter does not know the size of the next request in advance; a tightly calculated budget can therefore be exceeded by the size of one request, and that very request would be billed in full under the rule described above. Second, daily rate limits (TPD) are, depending on the tier and model, below the free allowance; a budget above the TPD limit would never be reached normally because the API would first reject the request with HTTP 429.

## Monitoring: The Dashboard Must Show 0.00

The platform’s Usage dashboard shows whether the calculation works out. Two views are sufficient:

- The **Usage** view counts all tokens, including those billed for free. It shows the workload’s full consumption.
- The **Costs** view (and the "Monthly spend" field in the project list) shows only paid tokens. It must remain at 0.00 permanently.

For more detail, group the Usage view by *Service tier*: tokens billed for free appear there as a separate item, "data sharing incentive tier." A monthly calendar reminder to check the dashboard completes the chain of guardrails, because OpenAI can end the program with 30 days’ notice, and from that day onward the same workload would continue at the standard rate.

## Sources

1.  [OpenAI Help Center: Sharing feedback, evaluation and fine-tuning data, and API inputs and outputs](https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai): authoritative program description with model groups, tier allowances, UTC reset, and the billing rule for requests that exceed the allowance.

2.  [OpenAI Developer Community: Extended: Free tokens on traffic shared with OpenAI](https://community.openai.com/t/good-news-extended-free-tokens-on-traffic-shared-with-openai/1241322): announcement of the program extension in April 2025, including the commitment to 30 days’ notice.

3.  [OpenAI Platform: Data sharing settings](https://platform.openai.com/settings/organization/data-controls/sharing): opt-in switch and enrollment status of your own organization (login required).

4.  [OpenAI Platform: Rate limits guide](https://platform.openai.com/docs/guides/rate-limits): explanation of the TPM, RPM, and TPD limits that apply alongside the free allowance.

5.  [OpenAI Platform: Pricing](https://platform.openai.com/docs/pricing): standard rates at which allowance overages are billed.
