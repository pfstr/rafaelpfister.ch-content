---
title: "A Serverless Newsletter: Cloudflare Workers, D1, and a Deploy Button"
navTitle: "Newsletter on Workers"
description: "A complete newsletter system (signup form, one-click unsubscribe, queued sending, compliance footer) runs serverless on your own Cloudflare account; even large lists stay within the free tier. The deploy button provisions database, schema, and CI in a single step; no command line required."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min to read"
themen:
  - "cloudflare-workers"
slug: "serverless-newsletter-cloudflare-workers-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/en/blog/serverless-newsletter-cloudflare-workers-d1"
aiPrompt: |
  You are my Cloudflare assistant. Help me get the newsletter template running on my Cloudflare account, step by step:
  1. Deploy via the Deploy to Cloudflare button (set ADMIN_TOKEN, FROM_NAME, FROM_EMAIL).
  2. Wire up my own email delivery in src/email.ts (implement sendEmail(), set the provider secret, adjust isEmailConfigured()) and verify the sending domain via SPF/DKIM/DMARC.
  3. Set SENDER_ADDRESS (postal address for the mandatory footer) and optionally PRIVACY_URL.
  4. Optionally enable double opt-in, Turnstile bot protection, RSS auto-send, and, if the provider offers a batch endpoint, sendEmailBatch() with a higher SEND_BATCH.
  Ask me for the values only I know (domain, email provider, postal address, feed URL).
---

# A Serverless Newsletter: Cloudflare Workers, D1, and a Deploy Button

Newsletter services bill by subscriber count, and the recipient list lives with the provider. The classic alternative (running a newsletter system on your own server) rarely fails because of the software; it fails because of operations: a server needs provisioning, patching, monitoring, and paying, all for a tool that might send one email a week.

Serverless fundamentally changes that calculation. A newsletter backend consists of a few HTTP endpoints, a small database, and a scheduled job: exactly the profile Cloudflare Workers and D1 are built for. I have implemented this as an open template: a complete newsletter system that lands on your own Cloudflare account in a single step via the **Deploy to Cloudflare button**. No command line, no server operations, and thanks to queued sending, even lists with thousands of recipients stay within the free tier. The source code is MIT-licensed on [GitHub](https://github.com/pfstr/newsletter-template); the PR to include it in the official Cloudflare template gallery is in progress.

[![Deploy to Cloudflare](../images/serverless-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![The template's hosted signup form](../images/serverless-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## What the template does

- **Signup**: a hosted signup page, an embeddable form for your own website, and a JSON endpoint
- **One-click unsubscribe**: RFC 8058 compliant, with an individual token per subscriber
- **Compliance built in**: every email automatically gets a footer with an unsubscribe link and postal address; consent and opt-out timestamps are stored (CAN-SPAM/GDPR)
- **Sending**: a protected page where you paste subject and HTML, send a test to yourself, then queue the campaign; a background job delivers in batches, with retries
- **Your data**: subscribers live in a D1 database on your account and can be exported at any time
- **Optional, off by default**: double opt-in, bot protection via Turnstile, and automatic delivery of new blog posts from your RSS feed

## Architecture: one Worker, one database

The entire system is a single Cloudflare Worker with two handlers: `fetch` for HTTP (routed with Hono) and `scheduled` for the cron trigger, plus a D1 database. There is no second service, no separate queue broker, no dedicated admin backend; even the send queue is just a D1 table.

| Route | Purpose |
| --- | --- |
| `GET /` | Hosted signup page |
| `GET /embed` | Transparent form for iframe embedding |
| `POST /api/subscribe` | Signup (CORS-open for your own website) |
| `GET /confirm` | Confirmation link for double opt-in |
| `GET/POST /unsubscribe` | Unsubscribe: confirmation page on GET, execution on POST (one-click per RFC 8058) |
| `GET /admin` | Send page (form) |
| `POST /api/send` | Queue a campaign, protected by admin token |

The data model comprises four tables: `subscribers` (email as primary key, name, status, unsubscribe and confirmation tokens, a JSON column for custom extra fields, plus timestamps for confirmation and opt-out), `campaigns` with subject, body, and per-send counters, `outbox` as the send queue (one row per recipient), and `sent_posts` for deduplicating the RSS send.

## Deployment without a command line

The most interesting part is not the code but the path to a running system. The Deploy to Cloudflare button reads the repository's Wrangler configuration and handles the complete setup: it clones the repository into your own GitHub account, provisions the D1 database, runs the schema migrations, and sets up CI so that every push deploys automatically. Since July 2025, the deploy flow additionally collects environment variables and secrets directly in the form: in this template's case the admin password (`ADMIN_TOKEN`), sender name and address, the double-opt-in switch, and the send batch size (`SEND_BATCH`).

The result after one click and one form: the signup page is live at `https://<worker-name>.workers.dev` and collecting subscribers. A terminal is never opened at any point.

## Collecting subscribers

For integration into your own website there are three paths, in ascending order of integration depth. The simplest: share the link to the hosted signup page. The most practical for site builders (WordPress, Webflow, Squarespace, Framer): a one-line iframe in any HTML embed block.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

If you want the form in your own design, post directly to the endpoint:

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Subscribe</button>
</form>
```

By default the form collects the email address and optionally a name. You define additional fields (company, country, …) in a single file (`src/fields.ts`); they automatically appear on both forms and are stored as JSON in the database.

## Sending: your own provider instead of a built-in vendor

For email delivery, the template makes a deliberate choice: it is **provider-agnostic**. The file `src/email.ts` contains a single `sendEmail()` adapter with a commented example for a generic HTTP API. Which delivery service you wire up there is your choice. No vendor is hard-coded, no registration with any particular service is required. Collecting subscribers already works entirely without delivery configuration; sending unlocks once the adapter is implemented and the provider secret is set. If your provider additionally offers a batch endpoint (one API call, many emails), you can add an optional `sendEmailBatch()` adapter in the same file; a commented example is included for that too.

You operate sending from the `/admin` page: paste subject and email HTML, send a test to your own address, then queue the campaign for all subscribers. The merge tags `{{unsubscribe_url}}`, `{{email}}`, and `{{name}}` are available in your emails.

The actual delivery happens in the background, following the transactional outbox pattern: `POST /api/send` writes the campaign and one row per recipient into the database and responds immediately. A minutely cron job then delivers `SEND_BATCH` emails per run, 40 by default: chosen so that every run stays within the Workers free plan's subrequest limits. Rows are claimed atomically, so overlapping runs can never double-send; failed deliveries are retried up to three times, and crashed runs are picked up again after ten minutes. And anyone who unsubscribes while their email is still queued no longer receives it: the opt-out also cancels messages that are already queued.

## Compliance is built in, not bolted on

Anyone who sends a newsletter is subject to anti-spam and privacy law: the US CAN-SPAM Act, GDPR and ePrivacy in the EU, the Unfair Competition Act in Switzerland. A substantial part of what newsletter services are paid for is exactly this compliance work. The template takes over the mechanical part of it:

- **Mandatory footer**: every campaign email automatically gets a footer with a working unsubscribe link and the sender's postal address (`SENDER_ADDRESS`); CAN-SPAM requires a physical address in commercial email. The send page warns as long as the address is missing.
- **List-Unsubscribe headers per RFC 8058** on every send: the native unsubscribe button in Gmail and Outlook, which Gmail and Yahoo have required from bulk senders since 2024. The app builds the headers; your provider adapter just passes them through.
- **Scanner-proof unsubscribe**: the unsubscribe link leads to a confirmation page with a single button. Corporate mail scanners that prefetch every link in an email can no longer unsubscribe anyone by accident; mail clients use the one-click POST directly.
- **Data minimization and proof**: an opt-out takes effect immediately, deletes name and extra fields, and is recorded with a timestamp, as are signup and double-opt-in confirmation. Consent can be evidenced later (GDPR accountability).
- **Privacy link**: with `PRIVACY_URL` set, a link to your privacy policy appears under the signup form.

What remains with the operator: truthful sender and subject lines, sending only to genuinely subscribed addresses, and domain authentication (SPF/DKIM/DMARC) with the delivery service. None of this is legal advice.

## Options: double opt-in, Turnstile, RSS automation

Three features are built in but disabled by default, so the system runs without configuration:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): new subscribers are stored as `pending` and only become active after clicking a confirmation link. For Switzerland (FADP) and the EU, this is the cleaner choice.
- **Bot protection** with Cloudflare Turnstile: set the site and secret key as variables, nothing more; the widget automatically appears on both forms, and the Worker verifies every signup server-side. Without a valid token, the signup is rejected.
- **RSS auto-send**: a cron job checks your blog feed (RSS 2.0 or Atom) every 15 minutes and automatically queues new posts for delivery. Two safeguards are built in: on the very first run, the existing feed is only recorded as a baseline (so the archive is not blasted out as an email flood), and every item ID is stored in `sent_posts`, so no post ever goes out twice.

## Limits

The template is deliberately minimal. On the free plan, the queued send delivers around 40 emails per minute by default; a campaign to 1,000 recipients takes about 25 minutes, which is irrelevant for a newsletter. On the paid Workers plan (10,000 subrequests per invocation instead of 50), `SEND_BATCH` can be raised into the hundreds; with a batch adapter (one API call, up to around 1,000 emails), even the free plan works through large lists in a few minutes. Deliverability depends, as with any system, on your own sending domain: SPF, DKIM, and DMARC must be verified with your chosen delivery service, otherwise the newsletter ends up in spam. And the single-opt-in default is the simplest start, but not the most conservative compliance option; that is what the switch is for.

On cost: Workers and D1 have generous free-tier quotas (including 100,000 requests per day) that a signup form and weekly sends to a small or medium list will not exhaust. If a limit is reached, Cloudflare throttles on the free plan instead of sending a bill.

## Try it

The source code including the deploy button is on [GitHub](https://github.com/pfstr/newsletter-template), along with the full documentation of the configuration variables.

[![GitHub: pfstr/newsletter-template](../images/serverless-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Sources

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template) — source code of the template (MIT) with deploy button and documentation.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/) — automatic provisioning of resources, repo cloning, and CI on deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/) — secrets and variables are collected in the deploy form since July 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/) — serverless SQLite, used here for subscribers, the send log, and RSS deduplication.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) — bot protection without captcha puzzles, optionally enabled in the template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058) — Signaling One-Click Functionality for List Email Headers; the basis of the native unsubscribe button in Gmail and Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) — subrequest limits per invocation (50 on the free plan, 10,000 on the paid plan); the send queue's batch size derives from these.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business) — obligations for commercial email, including the postal address and a working opt-out.
