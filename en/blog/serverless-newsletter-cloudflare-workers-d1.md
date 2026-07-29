---
title: "Run your own newsletter with Cloudflare Workers and D1"
navTitle: "Newsletter on Workers"
description: "The open template provides sign-up, unsubscribe, queue and database in your own Cloudflare account. A deploy button sets up Worker, D1 and CI without a local server."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min read"
themen:
  - "cloudflare-workers"
slug: "serverless-newsletter-cloudflare-workers-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/en/blog/serverless-newsletter-cloudflare-workers-d1"
---

# Run your own newsletter with Cloudflare Workers and D1

With a hosted newsletter service, the recipient list resides with the provider, and costs often rise with the number of subscribers. Running your own server provides more control, but entails ongoing work: updates, monitoring, backups and operating a system that may only send once a week.

For this lean use case, HTTP endpoints, a small database and a scheduled sending job are sufficient. Cloudflare Workers and D1 provide precisely these building blocks. My open template sets them up in your own account via a **Deploy-to-Cloudflare button**. No local command line or server requiring ongoing maintenance is needed. The MIT-licensed source code is available on [GitHub](https://github.com/pfstr/newsletter-template).

[![Deploy to Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![The template's hosted sign-up form](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## What the template can do

- **Sign-up**: a hosted sign-up page, an embeddable form for your own website and a JSON endpoint
- **One-click unsubscribe**: compliant with RFC 8058, with an individual token for each subscriber
- **Required information built in**: Every email automatically receives a footer with an unsubscribe link and postal address; consent and unsubscribe timestamps are stored
- **Sending**: On a protected page, you can enter a subject line and HTML, send a test email and queue the campaign; a background job sends in batches and retries failed attempts
- **Your own data**: Subscribers are held in a D1 database in your account and can be exported at any time
- **Optional, disabled by default**: Double opt-in, bot protection via Turnstile and automatic sending of new blog posts from the RSS feed

## Architecture: one Worker, one database

The entire system is a single Cloudflare Worker with two handlers: `fetch` for HTTP (routed with Hono) and `scheduled` for the cron trigger, plus a D1 database. There is no second service, no separate queue broker, no dedicated admin backend; even the sending queue is just a D1 table.

| Route | Function |
| --- | --- |
| `GET /` | Hosted sign-up page |
| `GET /embed` | Transparent form for embedding via iframe |
| `POST /api/subscribe` | Sign-up (CORS-enabled for your own website) |
| `GET /confirm` | Confirmation link for double opt-in |
| `GET/POST /unsubscribe` | Unsubscribe: confirmation page via GET, action via POST (one-click according to RFC 8058) |
| `GET /admin` | Sending page (form) |
| `POST /api/send` | Queue a campaign, protected by admin token |

The data model comprises four tables: `subscribers` (email as primary key, name, status, unsubscribe and confirmation tokens, a JSON column for custom additional fields, plus timestamps for confirmation and unsubscription), `campaigns` with subject, content and counters for each mailing, `outbox` as the sending queue (one row per recipient) and `sent_posts` for RSS sending deduplication.

## Deployment without a command line

The most interesting part is not the code, but the path to a running system. The Deploy-to-Cloudflare button reads the repository's Wrangler configuration and handles the entire setup: it clones the repository into your own GitHub account, provisions the D1 database, runs the schema migrations and sets up CI so that every push deploys automatically. Since July 2025, the deploy flow has also requested environment variables and secrets directly in the form: in this template's case, the admin password (`ADMIN_TOKEN`), sender name and address, the double-opt-in switch and the sending batch size (`SEND_BATCH`).

The result after one click and one form: the sign-up page is live at `https://<worker-name>.workers.dev` and collects subscribers. A terminal is never opened.

## Collecting subscribers

There are three ways to integrate it into your own website, in increasing order of integration depth. The simplest is to share the link to the hosted sign-up page. The most practical option for site builders (WordPress, Webflow, Squarespace, Framer) is a one-line iframe in any HTML embed block.

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
  <button>Abonnieren</button>
</form>
```

The form collects email by default and optionally a name. Define further fields (company, country, …) in a single file (`src/fields.ts`); they automatically appear on both forms and are stored as JSON in the database.

## Sending: your own provider instead of a built-in vendor

For email delivery, the template makes a deliberate choice: it is **provider-agnostic**. The file `src/email.ts` contains a single `sendEmail()` adapter with a commented example for a generic HTTP API. Which sending service you connect there is your choice. No provider is hard-wired, and registration with a particular service is not required. Collecting subscribers already works entirely without sending configuration; sending is enabled as soon as the adapter is implemented and the provider secret is set. If the provider also offers a batch endpoint (one API call, many emails), an optional `sendEmailBatch()` adapter can be added in the same file; a commented example is provided for that too.

Sending is managed via the `/admin` page: enter the subject and email HTML, send a test to your own address, then queue the campaign for all subscribers. The merge tags `{{unsubscribe_url}}`, `{{email}}` and `{{name}}` are available in emails.

The actual sending happens in the background, following the transactional outbox pattern: `POST /api/send` writes the campaign and one row per recipient to the database and responds immediately. A per-minute cron job then delivers `SEND_BATCH` emails per run, 40 by default: chosen so that each run remains within the subrequest limits of the Workers Free plan. Rows are claimed atomically, so overlapping runs can never send duplicates; failed deliveries are retried up to three times, and crashed runs are resumed after ten minutes. And anyone who unsubscribes while their email is still in the queue will no longer receive it: opting out also cancels messages that have already been queued.

## Unsubscribing and records are core features

Anyone sending a newsletter is subject to anti-spam and data-protection law: the US CAN-SPAM Act, the GDPR and ePrivacy rules in the EU, and the UWG in Switzerland. A significant part of what newsletter services are paid for is precisely meeting these obligations. The template handles the mechanical part:

- **Required footer**: Every campaign email automatically receives a footer with a working unsubscribe link and the sender's postal address (`SENDER_ADDRESS`); CAN-SPAM requires a physical address in commercial emails. The sending page warns while the address is missing.
- **List-Unsubscribe headers according to RFC 8058** on every mailing: the native unsubscribe button in Gmail and Outlook, which Gmail and Yahoo have required from bulk senders since 2024. The app assembles the headers; your own provider adapter only needs to pass them through.
- **Scanner-safe unsubscribe**: The unsubscribe link leads to a confirmation page with a single button. Corporate email scanners that pre-fetch every link in an email cannot accidentally unsubscribe anyone; email clients use the one-click POST directly.
- **Data minimisation and evidence**: An opt-out takes effect immediately, deletes the name and additional fields, and is recorded with a timestamp, as are sign-up and double-opt-in confirmation. Consent can therefore be evidenced later (GDPR accountability).
- **Privacy link**: With `PRIVACY_URL` set, a link to your own privacy policy appears beneath the sign-up form.

The operator remains responsible for truthful sender and subject lines, sending only to genuinely subscribed addresses, and domain authentication (SPF/DKIM/DMARC) with the sending service. None of this constitutes legal advice.

## Options: double opt-in, Turnstile, RSS automation

Three features are built in but disabled by default, so the system remains usable without configuration:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): New subscribers are stored as `pending` and become active only after clicking a confirmation link. This procedure is the more robust choice for Switzerland (FADP) and the EU.
- **Bot protection** with Cloudflare Turnstile: simply set the site and secret keys as variables; the widget automatically appears on both forms, and the Worker verifies each sign-up server-side. Sign-up is rejected without a valid token.
- **Automatic RSS sending**: A cron job checks your own blog feed (RSS 2.0 or Atom) every 15 minutes and automatically queues new posts for sending. Two safeguards are built in: on the very first run, the existing feed is merely marked as a baseline (so the archive is not sent as an email flood), and every post ID is recorded in `sent_posts`, so no post is sent twice.

## Limitations

The template is deliberately minimal. In the Free plan, queue sending delivers around 40 emails per minute by default; a campaign to 1,000 recipients therefore takes about 25 minutes, which does not matter for a newsletter. In the paid Workers plan (10,000 subrequests per invocation rather than 50), `SEND_BATCH` can be increased into the hundreds; with a batch adapter (one API call, up to around 1,000 emails), even the Free plan sends large lists in a few minutes. As with any system, deliverability depends on your own sender domain: SPF, DKIM and DMARC must be verified with the chosen sending service, otherwise the newsletter will land in spam. And the single-opt-in default is the simplest starting point, but not the most conservative compliance option; that is what the switch is for.

As for costs: Workers and D1 have generous Free Tier allowances (including 100,000 requests per day), which a sign-up form and weekly mailings to a small to medium-sized list will not exhaust. If a limit is reached, Cloudflare throttles on the Free plan rather than issuing a bill.

## Try it out

The source code, including the deploy button, is available on [GitHub](https://github.com/pfstr/newsletter-template); the full documentation of the configuration variables is also available there.

[![GitHub: pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Sources

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template): Template source code (MIT) with deploy button and documentation.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/): automatic resource provisioning, repository cloning and CI during deployment.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/): secrets and variables have been requested in the deployment form since July 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/): serverless SQLite, used here for subscribers, sending log and RSS deduplication.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/): bot protection without CAPTCHA puzzles, optionally enabled in the template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058): Signalling One-Click Functionality for List Email Headers; the basis for the native unsubscribe button in Gmail and Outlook.

7.  [Workers limits](https://developers.cloudflare.com/workers/platform/limits/): subrequest limits per invocation (50 in the Free plan, 10,000 in the paid plan); the queue sending batch size is derived from these.

8.  [FTC: CAN-SPAM Act Compliance Guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business): obligations for commercial emails, including a postal address and a working opt-out.
