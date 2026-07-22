---
title: "A Serverless Newsletter: Cloudflare Workers, D1, and a Deploy Button"
navTitle: "Newsletter on Workers"
description: "A complete newsletter system (signup form, one-click unsubscribe, send page) runs serverless on your own Cloudflare account and stays within the free tier for small and medium lists. The deploy button provisions database, schema, and CI in a single step; no command line required."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "7 min to read"
themen:
  - "cloudflare-workers"
slug: "serverless-newsletter-cloudflare-workers-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/en/blog/serverless-newsletter-cloudflare-workers-d1"
aiPrompt: |
  You are my Cloudflare assistant. Help me get the newsletter template running on my Cloudflare account, step by step:
  1. Deploy via the Deploy to Cloudflare button (set ADMIN_TOKEN, FROM_NAME, FROM_EMAIL).
  2. Wire up my own email delivery in src/email.ts (implement sendEmail(), set the provider secret, adjust isEmailConfigured()) and verify the sending domain via SPF/DKIM.
  3. Optionally enable double opt-in, Turnstile bot protection, and the RSS auto-send.
  Ask me for the values only I know (domain, email provider, feed URL).
---

<!-- AFTER cloudflare/templates PR #1064 IS MERGED: re-add the gallery link https://github.com/cloudflare/templates/tree/main/newsletter-template and the live demo https://newsletter-template.templates.workers.dev (intro, "Try it" section, sources; DE + EN). Both URLs were still 404 at publish time, hence removed. -->

# A Serverless Newsletter: Cloudflare Workers, D1, and a Deploy Button

Newsletter services bill by subscriber count, and the recipient list lives with the provider. The classic alternative (running a newsletter system on your own server) rarely fails because of the software; it fails because of operations: a server needs provisioning, patching, monitoring, and paying, all for a tool that might send one email a week.

Serverless fundamentally changes that calculation. A newsletter backend consists of a few HTTP endpoints, a small database, and a scheduled job: exactly the profile Cloudflare Workers and D1 are built for. I have implemented this as an open template: a complete newsletter system that lands on your own Cloudflare account in a single step via the **Deploy to Cloudflare button**. No command line, no server operations, and within the free tier for small and medium lists. The source code is MIT-licensed on [GitHub](https://github.com/pfstr/newsletter-template); the PR to include it in the official Cloudflare template gallery is in progress.

<p class="badge-row"><a href="https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template"><img src="/images/deploy-to-cloudflare.svg" alt="Deploy to Cloudflare" width="184" height="39"></a></p>

![The template's hosted signup form](../images/newsletter-template-signup.png)

## What the template does

- **Signup**: a hosted signup page, an embeddable form for your own website, and a JSON endpoint
- **One-click unsubscribe**: RFC 8058 compliant, with an individual token per subscriber
- **Sending**: a protected page where you paste subject and HTML, send a test to yourself, then send to the list
- **Your data**: subscribers live in a D1 database on your account and can be exported at any time
- **Optional, off by default**: double opt-in, bot protection via Turnstile, and automatic delivery of new blog posts from your RSS feed

## Architecture: one Worker, one database

The entire system is a single Cloudflare Worker with two handlers: `fetch` for HTTP (routed with Hono) and `scheduled` for the cron trigger, plus a D1 database. There is no second service, no queue broker, no separate admin backend.

| Route | Purpose |
| --- | --- |
| `GET /` | Hosted signup page |
| `GET /embed` | Transparent form for iframe embedding |
| `POST /api/subscribe` | Signup (CORS-open for your own website) |
| `GET /confirm` | Confirmation link for double opt-in |
| `GET/POST /unsubscribe` | Unsubscribe, one-click per RFC 8058 |
| `GET /admin` | Send page (form) |
| `POST /api/send` | Campaign send, protected by admin token |

The data model comprises three tables: `subscribers` (email as primary key, name, status, unsubscribe and confirmation tokens, plus a JSON column for custom extra fields), `campaigns` as a send log, and `sent_posts` for deduplicating the RSS send.

## Deployment without a command line

The most interesting part is not the code but the path to a running system. The Deploy to Cloudflare button reads the repository's Wrangler configuration and handles the complete setup: it clones the repository into your own GitHub account, provisions the D1 database, runs the schema migrations, and sets up CI so that every push deploys automatically. Since July 2025, the deploy flow additionally collects environment variables and secrets directly in the form: in this template's case the admin password (`ADMIN_TOKEN`), sender name and address, and the double-opt-in switch.

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

For email delivery, the template makes a deliberate choice: it is **provider-agnostic**. The file `src/email.ts` contains a single `sendEmail()` adapter with a commented example for a generic HTTP API. Which delivery service you wire up there is your choice. No vendor is hard-coded, no registration with any particular service is required. Collecting subscribers already works entirely without delivery configuration; sending unlocks once the adapter is implemented and the provider secret is set.

Sending itself happens on the `/admin` page: paste subject and email HTML, send a test to your own address, then send to all subscribers. The merge tags `{{unsubscribe_url}}`, `{{email}}`, and `{{name}}` are available in your emails. Every message carries the List-Unsubscribe headers per RFC 8058: Gmail and Outlook show their native unsubscribe button, and unsubscribing works with one click, no login.

## Options: double opt-in, Turnstile, RSS automation

Three features are built in but disabled by default, so the system runs without configuration:

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`): new subscribers are stored as `pending` and only become active after clicking a confirmation link. For Switzerland (FADP) and the EU, this is the cleaner choice.
- **Bot protection** with Cloudflare Turnstile: set the site and secret key as variables, nothing more; the widget automatically appears on both forms, and the Worker verifies every signup server-side. Without a valid token, the signup is rejected.
- **RSS auto-send**: a cron trigger checks your blog feed (RSS 2.0 or Atom) every 15 minutes and automatically sends new posts to the list. Two safeguards are built in: on the very first run, the existing feed is only recorded as a baseline (so the archive is not blasted out as an email flood), and every item ID is stored in `sent_posts`, so no post ever goes out twice.

## Limits

The template is deliberately minimal, and its limits are documented rather than hidden. The campaign send is a simple loop and therefore carries a few hundred recipients per send (Workers subrequest limits); for larger lists, you move sending to Cloudflare Queues. Deliverability depends, as with any system, on your own sending domain: SPF and DKIM must be verified with your chosen delivery service, otherwise the newsletter ends up in spam. And the single-opt-in default is the simplest start, but not the most conservative compliance option; that is what the switch is for.

On cost: Workers and D1 have generous free-tier quotas (including 100,000 requests per day) that a signup form and weekly sends to a small or medium list will not exhaust. If a limit is reached, Cloudflare throttles on the free plan instead of sending a bill.

## Try it

The source code including the deploy button is on [GitHub](https://github.com/pfstr/newsletter-template), along with the full documentation of the configuration variables.

<p class="badge-row"><a href="https://github.com/pfstr/newsletter-template"><img src="/images/github-newsletter-template.svg" alt="GitHub: pfstr/newsletter-template" width="323" height="28"></a></p>

## Sources

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template) — source code of the template (MIT) with deploy button and documentation.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/) — automatic provisioning of resources, repo cloning, and CI on deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/) — secrets and variables are collected in the deploy form since July 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/) — serverless SQLite, used here for subscribers, the send log, and RSS deduplication.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) — bot protection without captcha puzzles, optionally enabled in the template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058) — Signaling One-Click Functionality for List Email Headers; the basis of the native unsubscribe button in Gmail and Outlook.

7.  [Cloudflare Queues](https://developers.cloudflare.com/queues/) — the documented path for sending to larger lists.
