---
title: "APIs and integrations: REST, tokens, and automation"
blatt: "apis"
description: "What administrators need to know about APIs: the basic REST pattern, authentication with keys, tokens, and OAuth, rate limits and versioning, local device APIs versus cloud APIs, and the first test with curl."
fakten:
  - { label: "Basic pattern", wert: "REST over HTTPS: endpoints, methods, JSON" }
  - { label: "Authentication", wert: "API keys, bearer tokens, OAuth 2.0", href: "https://oauth.net/2/" }
  - { label: "Description", wert: "OpenAPI specification (Swagger)", href: "https://swagger.io/specification/" }
  - { label: "Operational factors", wert: "Rate limits, versioning, idempotency" }
  - { label: "Variants", wert: "Cloud APIs, local device APIs, webhooks" }
  - { label: "Tools", wert: "curl, jq, Postman/Bruno" }
werbung: ["newsletter"]
ctaThemen: ["cloudflare-workers", "smart-home-iot"]
---

# APIs and integrations: REST, tokens, and automation

Whether a smart home device, a mail platform, or a serverless service: the management interface is only the facade, and behind it there is almost always an API. Addressing that API directly makes it possible to automate tasks, build integrations, and understand failure patterns that remain invisible in the GUI.

## The basic REST pattern

Most APIs follow the same pattern: **endpoints** as URLs (`/api/v1/devices`), **HTTP methods** as verbs (GET reads, POST creates, PUT/PATCH modifies, DELETE removes), and **JSON** as the data format. The response status codes carry half the diagnosis: `401` means not authenticated, `403` means authenticated but not authorized, `429` means too many requests, and `5xx` means the problem lies with the provider. Being able to interpret these four codes resolves the majority of integration faults without a support ticket.

## Authentication: from key to OAuth

The ladder of methods, from simple to robust: **API keys** (a static secret in the header, simple, but without expiry or scope), **bearer tokens** (short-lived, often issued by a login or token endpoint), and **OAuth 2.0** (delegated, scoped access, the standard on cloud platforms). Operational rules apply regardless of the method: secrets belong in a vault or secret store rather than in scripts, every integration gets its own credential so that revocation causes no collateral damage, and expiry dates are monitored the same way certificates are.

## Local or cloud: an architectural question

Device integrations come in two flavors. The **local API** talks directly to the device on the local network: fast, capable of running offline, and private, but often less well documented and with quirks around tokens and keys. The **cloud API** runs through the vendor's servers: convenient and documented, but dependent on internet connectivity, an account, and the continued existence of the service. For anything operationally critical, the rule of thumb is: local where possible, cloud where necessary. **Webhooks** reverse the direction; the service calls into the local infrastructure when events occur, which avoids polling but requires a reachable, properly secured endpoint.

## The first test belongs in the shell

```bash
curl -s -H "Authorization: Bearer $TOKEN" https://api.example.ch/v1/status | jq .
```

