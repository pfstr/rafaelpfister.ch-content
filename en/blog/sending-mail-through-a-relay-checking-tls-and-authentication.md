---
title: "Sending mail through a relay: Checking TLS and authentication"
navTitle: "Relay: Check TLS"
description: "A one-pager for Application Managers whose applications send mail through a relay: Which three application settings matter (port, TLS mode, authentication), what the options are called in common environments, and how a single test email’s Received header proves that the connection is actually encrypted and authenticated."
date: "2026-08-28"
kategorie: "SMTP and mail flow"
timeToRead: "5 min read"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "sending-mail-through-a-relay-checking-tls-and-authentication"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
url: https://rafaelpfister.ch/en/blog/sending-mail-through-a-relay-checking-tls-and-authentication
translationSourceHash: 51d48e038c5eb870c77828f954ce1ad1d27bc4758889cb492c872eeaede04d9e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:29:04.068Z
translationReview: automatic
---

# Sending mail through a relay: Checking TLS and authentication

Many applications do not send email to the internet themselves, but instead submit it to an internal relay: the ERP sends its order confirmations, monitoring sends its alerts, and the ticketing system sends its notifications. The mail team operates the relay; the Application Manager is responsible for the application side. During an audit or protection requirements analysis, the question therefore lands with them: Does the application connect to the relay using encryption, and does it authenticate properly?

The answer can be found in two places that require neither a mail tool nor access to the relay: in the SMTP configuration of the application itself and in the header of a single test email. The mail team is responsible for what the relay itself offers and how it encrypts email onward to the recipient; on the application side, it is sufficient to document that portion of the path.

## Where to find the settings

Depending on the application, the SMTP configuration can be found in one of three places: in the administration interface (usually under “Email,” “Notifications,” “SMTP,” or “Outgoing Server”), in a configuration file, or in deployment environment variables (typically `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` and variants). The same information is always required: server name, port, an encryption option, and credentials.

## The three settings that matter

**First, port and TLS mode.** Both must match, because the selection values represent two different methods: With STARTTLS, the connection begins in cleartext and then switches to TLS; with implicit TLS (usually called “SSL/TLS” or “SSL” in user interfaces), it is encrypted from the start.

| Port | TLS setting in the application | Assessment |
|---|---|---|
| 587 | STARTTLS | Target state for submission by applications |
| 465 | SSL/TLS (implicit) | also acceptable |
| 25 | none or STARTTLS | common for relays with IP allowlisting; still enable the TLS setting if the relay offers STARTTLS |
| any | “None” | Finding: mail is sent in cleartext |
| any | “TLS if available” / opportunistic | Finding: silently falls back to cleartext if there is a problem; switch to enforced TLS |

A mismatch (such as “SSL/TLS” on port 587) causes connection failures, not unnoticed cleartext. The risky settings are the last two rows of the table, because the application sends unencrypted email without an error message in those cases.

**Second, certificate validation.** Many applications provide an option such as “Do not validate certificate,” “Allow insecure,” or `verify=false`, which is often enabled during implementation projects because the relay uses an internal certificate. The connection remains encrypted, but the application accepts any endpoint. If the option is enabled, it must be recorded as a finding in the report; the proper solution is to trust the internal CA rather than disable validation.

**Third, authentication.** Relays support two models: SMTP AUTH with a username and password, or IP allowlisting without an account. The applicable option is specified in the relay approval from the mail team. For SMTP AUTH, three items belong on the checklist: Authentication uses a dedicated application service account (not a personal account that will be disabled when its owner leaves), the password is stored as a secret rather than in cleartext in a configuration file, and the TLS option is enabled, because common methods such as PLAIN and LOGIN otherwise transmit credentials in cleartext.

## What the settings are called in common environments

| Environment | Encryption | Authentication |
|---|---|---|
| Admin interfaces (ERP, monitoring, appliances) | “Encryption” dropdown: None / STARTTLS / SSL-TLS | Username/password fields; blank = no authentication |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` plus `mail.smtp.starttls.required=true`; for port 465, `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (enables STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` or `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` for 587, `'ssl'` for 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` after establishing the connection, or `SMTP_SSL` for 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; port 587: `secure:false` plus `requireTLS:true` | `auth: {user, pass}` |

Experience shows that two points from this table are the most frequent findings: In Java, `starttls.enable` alone enables only opportunistic TLS; only `starttls.required` prevents fallback to cleartext. In Nodemailer, `secure:false` does not mean “unencrypted,” but rather “no implicit TLS”; without `requireTLS:true`, STARTTLS also remains opportunistic.

## Cross-check: a test email and its Received header

The configuration states the target state, but it does not prove what happens on the wire. The proof is in the Received header that the relay adds when receiving each email. A test email sent from the application to your own mailbox is sufficient; display the message header there (Outlook: File, Properties, Internet headers; Gmail: Show original) and read the bottommost Received line, because headers grow from bottom to top:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

The keyword after `with` is the short form of the test result. The identifiers are standardized (IANA registry “Mail Transmission Types”):

| Identifier | Meaning | Assessment |
|---|---|---|
| `SMTP` / `ESMTP` | unencrypted, without authentication | Action required if TLS is mandated |
| `ESMTPS` | TLS, without authentication | acceptable with IP allowlisting |
| `ESMTPA` | authenticated, but without TLS | critical: credentials were transmitted in cleartext |
| `ESMTPSA` | TLS and authenticated | Target state for SMTP AUTH |

Postfix and Exchange add the TLS version and cipher in parentheses, which also makes it possible to identify outdated protocol versions. For analyzing longer headers with multiple hops, the [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) on this website saves you the manual effort; it runs entirely locally in the browser, and the header never leaves your computer.

If the header remains unclear or an upstream load balancer rewrites the connection marker, it is time to contact the mail team: The relay log records for each submission whether TLS was negotiated and which account the application used to authenticate.

## Short checklist for the audit report

1. SMTP configuration of the application found (interface, configuration file, or environment variables) and documented.
2. Port and TLS mode match (587/STARTTLS or 465/SSL-TLS); no “None” or “TLS if available” setting.
3. Certificate validation enabled; any enabled “Do not validate certificate” setting is recorded as a finding.
4. Authentication model clarified: SMTP AUTH with a service account and secret storage, or IP allowlisting according to the relay approval.
5. Received header of the test email shows `ESMTPSA` (with an account) or `ESMTPS` (with IP allowlisting); `ESMTPA` and `ESMTP` are findings.
6. If encryption through to the recipient is required: address it as a requirement to the mail team, because the path beyond the relay lies outside the application.

## Sources

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): defines STARTTLS and switching the cleartext connection to TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): defines SMTP AUTH and methods such as PLAIN and LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): recommends implicit TLS on port 465 for client submission.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): registry of `with` identifiers in the Received header (ESMTPS, ESMTPA, ESMTPSA).

5.  [Jakarta Mail: Package com.sun.mail.smtp](https://jakarta.ee/specifications/mail/2.1/apidocs/jakarta.mail/com/sun/mail/smtp/package-summary): documents the properties mail.smtp.starttls.enable, starttls.required, and ssl.enable.
