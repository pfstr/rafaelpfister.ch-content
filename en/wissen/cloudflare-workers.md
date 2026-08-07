---
title: "Cloudflare Workers: serverless at the network edge"
blatt: "cloudflare-workers"
description: "An overview of Cloudflare's serverless platform: code at the edge instead of on servers, the V8 isolate model, storage services from KV to D1 and R2, bindings and Wrangler, and the limits of the model."
fakten:
  - { label: "Provider", wert: "Cloudflare" }
  - { label: "Model", wert: "Serverless on V8 isolates, worldwide at the edge" }
  - { label: "Documentation", wert: "developers.cloudflare.com/workers", href: "https://developers.cloudflare.com/workers/" }
  - { label: "Languages", wert: "JavaScript/TypeScript, WebAssembly" }
  - { label: "Storage services", wert: "KV, D1 (SQLite), R2 (object storage), Durable Objects, Queues" }
  - { label: "Tooling", wert: "Wrangler (CLI) for development and deployment" }
werbung: ["newsletter"]
ctaThemen: ["cloudflare-workers"]
---

# Cloudflare Workers: serverless at the network edge

Cloudflare Workers does not run code on a server in one region but within Cloudflare's global network, at the point closest to the caller. For small services, APIs, and automations, this results in infrastructure without server operations: no patching, no capacity planning, and usage-based billing.

## The isolate model

Workers run as **V8 isolates**, the same sandboxes that separate browser tabs. Isolates start within milliseconds, which is why the model largely avoids cold starts, the classic annoyance of other serverless platforms. The flip side consists of deliberate limits: short CPU time windows per request, limited memory, no traditional file system, and no arbitrary runtimes, but web standard APIs instead (fetch, Request/Response, Crypto). Workers are therefore suited to request-driven logic, not to long-running processes.

## State: KV, D1, R2, and Durable Objects

Because isolates are ephemeral, state resides in attached services: **KV** as a globally distributed key-value store for configuration and caches (eventually consistent), **D1** as a serverless SQLite database for relational data, **R2** as S3-compatible object storage without egress fees, **Durable Objects** for strongly consistent state with a single-instance guarantee, and **Queues** for asynchronous processing. Access runs through **bindings**: the Worker receives its resources as objects in the runtime environment instead of through connection strings in the code.

## Development and operations

The platform's tool is **Wrangler**: local development with a simulated runtime, deployment within seconds, and management of secrets, logs, and cron triggers (scheduled executions without a request). Configuration is declarative per project; routes bind Workers to domains or paths, which allows individual endpoints of an existing website to be backed by logic while the rest remains static.

## Assessment

Workers compete less with full-scale cloud platforms than with a small self-run server: for form backends, newsletter sign-ups, webhooks, proxies, and scheduled jobs, a Worker together with D1 or KV replaces an entire VPS and its maintenance. The limits are set by CPU quotas, the absence of persistent processes, and the tie to the Cloudflare ecosystem.
