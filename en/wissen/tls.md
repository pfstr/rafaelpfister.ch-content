---
title: "TLS and Certificates: Trust in Mail and Directory Operations"
blatt: "tls"
description: "How TLS secures connections and why certificates are the most common cause of outages: handshake, certificate chains and truststores, SAN instead of CN, LDAPS and StartTLS, renewal without downtime."
fakten:
  - { label: "Full name", wert: "Transport Layer Security (successor to SSL)" }
  - { label: "Purpose", wert: "Encryption, integrity, and server authentication at the transport layer" }
  - { label: "Current version", wert: "TLS 1.3 (2018), TLS 1.2 still widespread" }
  - { label: "Standard", wert: "RFC 8446 (TLS 1.3)", href: "https://datatracker.ietf.org/doc/html/rfc8446" }
  - { label: "Certificates", wert: "X.509 v3, RFC 5280", href: "https://datatracker.ietf.org/doc/html/rfc5280" }
  - { label: "Use in mail", wert: "SMTP StartTLS/465 · LDAPS 636 · HTTPS management 443" }
  - { label: "Public validity", wert: "max. 398 days, reduction to 47 days by 2029 adopted", href: "https://cabforum.org/working-groups/server/baseline-requirements/documents/" }
  - { label: "Tools", wert: "openssl s_client, openssl x509, certutil" }
werbung: ["tools", "newsletter"]
ctaThemen: ["e-mail-verschluesselung", "smtp-mailflow"]
---

# TLS and Certificates: Trust in Mail and Directory Operations

Hardly any outage is as predictable as an expired certificate, and hardly any hits so many services at the same time: SMTP delivery, LDAPS connections, management interfaces, and webhooks all depend on the same mechanism. Once TLS and the certificate chain have been understood properly, the annual renewal loses its terror and failure patterns become recognizable at a glance.

## What happens in the handshake

After the TCP connection has been established, client and server negotiate the TLS session: protocol version, cipher, and above all identity. The server presents its **certificate**, and the client checks three things: does the **hostname** match the names in the certificate? Is the **validity period** current? Does the **chain** lead through intermediate certificates to a root CA that the client trusts? If one of the three fails, the connection breaks off, and it does so before a single byte of payload flows.

## Chain and truststore: the most common source of errors

A certificate rarely comes alone. Besides its own **leaf certificate**, the server must also deliver the **intermediate certificates**; only the **root CA** resides in the client's **truststore**. The two classics in operations: a server does not deliver the intermediates (browsers forgive this thanks to caches, appliances and scripts do not), or an appliance does not know the internal corporate CA and therefore rejects LDAPS or an imported certificate. Every integration with internal services therefore comes with the question: is the root CA present in the client's truststore?

For the hostname, only the **Subject Alternative Name (SAN)** counts today; the Common Name is mere decoration. A certificate for an appliance should cover all names under which it is addressed, including cluster and management names.

## StartTLS or implicit TLS

Both paths lead to the same encryption. **Implicit TLS** starts the session immediately (LDAPS on 636, HTTPS on 443, SMTPS on 465), whereas **StartTLS** upgrades an existing cleartext connection (SMTP on 25/587, LDAP on 389). Operationally, implicit TLS is often more robust because a glance at the port is enough and there is no room for downgrades. For SMTP between foreign servers, StartTLS remains opportunistic; it becomes binding only with MTA-STS or DANE.

## Renewing without downtime

Certificate renewal can be planned: inventory the expiry dates (those of the appliances as well, not just the web servers), generate new key pairs in good time, deliver the chain along with certificates from internal CAs, and check every affected service after the import. Public certificates today are valid for a maximum of 398 days; the CA/Browser Forum decisions shorten this in stages down to 47 days in 2029. By then at the latest, manual renewal is dead and automation (ACME or vendor-specific paths) is mandatory.

## The one-line check

```bash
openssl s_client -connect mail.example.ch:636 -servername mail.example.ch 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName -enddate
```

