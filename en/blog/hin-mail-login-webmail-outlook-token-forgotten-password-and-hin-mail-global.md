---
title: "HIN Mail Login: Webmail, Outlook Token, Forgotten Password, and HIN Mail Global"
navTitle: "HIN Mail Login"
description: "Where and how to sign in to HIN Mail: webmail at webmail.hin.ch, mail tokens for Outlook and smartphones, login without the HIN Client using an SMS code or Authenticator app, password reset, and opening HIN Mail Global as a recipient without a HIN account."
date: "2026-10-05"
kategorie: "HIN Gateway"
timeToRead: "7 min read"
themen:
  - hin-gateway
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - hin-plattformerneuerung-2026
  - hin-mailgateway-backup-disaster-recovery
slug: "hin-mail-login-webmail-outlook-token-forgotten-password-and-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
url: https://rafaelpfister.ch/en/blog/hin-mail-login-webmail-outlook-token-forgotten-password-and-hin-mail-global
translationSourceHash: 8f863faa139fd22182bf59e849447e241e339e4fcc4e6a073e6c1a89344f8a06
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:38:44.904Z
translationReview: required
---

# HIN Mail Login: Webmail, Outlook Token, Forgotten Password, and HIN Mail Global

A search for “HIN Mail Login” usually relates to one of three situations: You are a HIN member and want to read your emails in a browser, you want to set up HIN Mail in Outlook or on your smartphone, or, as a patient or external organization, you have received an encrypted HIN email but do not have a HIN account. The sign-in process differs in each case.

| Situation | Address | Username | Password |
|---|---|---|---|
| Webmail in the browser | `webmail.hin.ch` | HIN ID | HIN password, secured by HIN Client, SMS code, or Authenticator app |
| Outlook, Thunderbird, Apple Mail, smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | Mail token from `apps.hin.ch` |
| Administration, token, initialization password | `apps.hin.ch` | HIN ID | HIN password plus second factor |
| Recipients without a HIN account (HIN Mail Global) | Attachment `secure-email.html` or link in the email | Mobile number | SMS code |

## HIN Webmail: Sign in in the browser

You can access webmail at `https://webmail.hin.ch`. Sign in with your HIN ID (the HIN login name, for example, `hmuster`) and your HIN password. One of the following methods provides the second factor:

- **HIN Client:** If the client is running on the workstation and you are signed in, webmail opens without any further prompt. With HIN Client 4.0, you sign in directly in the browser: enter the HIN-protected address and you are redirected to sign in.
- **SMS code (mTAN):** After entering your password, you receive a code at the stored mobile number. This method is suitable for devices without HIN Client, such as a personal laptop.
- **HIN Authenticator App:** The app (App Store and Google Play, “HIN Authenticator”) confirms the sign-in on your smartphone.

The SMS code and Authenticator app must be activated in advance. If neither is available and HIN Client is unavailable, you cannot sign in to webmail. Therefore, set up at least one alternative while the client is still working.

In webmail, you can compose and manage emails, maintain contacts and address books, search the HIN participant directory, and create multiple signatures. A bar beneath the username shows the available mailbox storage; if it is red, you need to delete emails or expand storage.

## HIN Mail in Outlook and on smartphones: Sign in with a mail token

Email clients do not sign in with the HIN password but with a **mail token**. This is the most common reason Outlook repeatedly asks for a password: the HIN password is rejected during IMAP sign-in.

To generate the token:

1. Open `https://apps.hin.ch` and sign in with your HIN ID (HIN Client, SMS code, or Authenticator app).
2. Select the “HIN Mail” section.
3. Click “Add device” and select “Mail Clients” as the device type.
4. Copy the displayed token and enter it as the password in your email client.

Server settings:

| Setting | Incoming mail (IMAP) | Outgoing mail (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Encryption | SSL/TLS | STARTTLS |
| Username | HIN ID | HIN ID |
| Password | Mail token | Mail token |
| Authentication | normal | Enable “Outgoing server requires authentication” |

The username is the HIN ID, not the email address. Outlook suggests the email address during automatic setup; therefore, you must configure it manually through “Advanced Options” or “Manual Setup.”

The token is linked to your HIN identity and can be revoked and regenerated at any time in `apps.hin.ch`. A separate token is recommended for each device: if a smartphone is lost, you only block that device, while Outlook on the practice PC continues to work. Once the token has been set up, Outlook no longer needs HIN Client to retrieve mail.

## First sign-in without HIN Client: Activate your identity

New HIN identities can be activated without HIN Client. You need the username and initialization password from the HIN documents.

1. Open the activation link in the Service Center (`servicecenter.hin.ch`).
2. Enter the username and initialization password.
3. Set your own password: at least 10 characters with numbers, uppercase and lowercase letters, and special characters.
4. Set up a second factor immediately (SMS code or Authenticator app).
5. Then sign in at `apps.hin.ch`.

You have 10 minutes to complete activation. After that, the initialization password is reset for security reasons, and you must request a new one.

## Forgot your HIN password

The reset process depends on whether an alternative sign-in method is set up.

**With SMS code or Authenticator app:**

1. Sign in at `apps.hin.ch` using the alternative method.
2. Select the “HIN Client” tab and generate a new initialization password.
3. In HIN Client, select “Registration,” then enter the login name and new initialization password.
4. Set a new password and confirm “Register new HIN identity.”

**Without an alternative method and without a signed-in device,** the only option is HIN Support by phone (0848 830 740, Monday through Friday, 8:00 a.m. to 6:00 p.m.). Write down the new initialization password before closing the window.

## HIN Mail Global: Open encrypted email without a HIN account

Medical practices, hospitals, and laboratories send messages to patients or organizations without HIN membership through HIN Mail Global. The email arrives in your inbox as a regular email, but its content is encrypted in attachment `secure-email.html` or behind the “Read encrypted message” link. There is no HIN login in the usual sense; you sign in using your mobile number.

**The first time:**

1. Download and open attachment `secure-email.html` (or click the link in the email). With webmail services such as Gmail, Bluewin, or GMX, save it first and then open it.
2. Select your language and enter your mobile number.
3. Enter the code received by SMS. The message is displayed.

**For subsequent messages,** simply open the attachment; the SMS code is automatically sent to the registered number.

Please note:

- The message can be accessed through the link for **30 days**, after which access expires. Save or print important content within this period.
- Only the original recipient can open forwarded messages.
- You can reply in encrypted form.
- You can mark a device as trusted; the SMS prompt is then skipped for one year.
- If no code arrives within 15 minutes, you can request a new one.
- The two current versions of Chrome, Safari, Edge, and Firefox are supported.

For issues with HIN Mail Global, contact HIN Mail Support at +41 58 670 48 69. Only the sender can answer questions about the content of the message.

## Common HIN Mail login errors

| Symptom | Cause | Solution |
|---|---|---|
| Outlook repeatedly asks for the password | HIN password entered instead of mail token | Generate a token in `apps.hin.ch` and enter it as the password |
| Outlook sign-in fails despite token | Email address used as username | Use the HIN ID as the username |
| Sending fails, receiving works | SMTP authentication is not enabled, or the port or encryption is incorrect | Use port `587` with STARTTLS and enable authentication |
| Webmail requests a code that never arrives | Mobile number is outdated or SMS code is not activated | Use the Authenticator app or sign in through HIN Client and update the number |
| Initialization password is rejected | The 10-minute period has expired | Request a new initialization password |
| Token worked but suddenly no longer does | Token revoked in `apps.hin.ch` or device removed | Generate a new token for this device |
| HIN Mail Global: Link no longer opens anything | The 30-day period has expired | Ask the sender to resend it |

## HIN Client 4.0: What changes with sign-in

HIN will automatically distribute HIN Client 4.0 by October 26, 2026; anyone who does not receive an automatic update can download it from `download.hin.ch`. The existing password remains valid and is confirmed once when you first sign in with the new version. You then sign in to HIN-protected services such as webmail directly in the browser.

Version 4.0 requires Windows 11 with TPM or macOS 14 with a Security Chip. Devices that do not meet these requirements can no longer use the client. Webmail access remains possible through an SMS code or Authenticator app, and Outlook can retrieve mail using the mail token. Details on the platform renewal deadlines are available in the article [HIN Platform Renewal 2026](/blog/hin-plattformerneuerung-2026).

## Medical practices and hospitals with their own mail server

Organizations with their own HIN Mailgateway do not sign in to HIN Mail. The mailboxes are hosted on their own Exchange or in Exchange Online, and the gateway (a SEPPmail appliance) encrypts mail traveling to and from the HIN community. Sign-in issues there concern the organization's own email system, not the HIN platform. This gateway is being replaced by the new HIN Gateway (“Stargate”).

<aside class="offer-box">
  <span class="offer-box__tag">Free assessment</span>
  <p><strong>Do you operate your own HIN Mailgateway?</strong> I will review your environment and tell you what needs to be done before migrating to “Stargate.”</p>
  <a class="offer-box__cta" href="/stargate">Register now</a>
</aside>

## Sources

1.  [HIN Support: HIN Mail and Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Overview with the webmail address and instructions for Outlook, Apple Mail, Thunderbird, iOS, and Android.

2.  [HIN Support: How does webmail work?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Webmail features and storage display.

3.  [HIN Support: Mail Token Service guide for Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Token generation in apps.hin.ch, IMAP and SMTP servers, ports, and encryption.

4.  [HIN Support: What is a token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): Mail tokens and access tokens, linked to the HIN identity.

5.  [HIN Support: Activate HIN identity (without HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Activation in the Service Center, 10-minute period, password rules, second factor.

6.  [HIN Support: Forgot password](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Reset using an initialization password and phone support without an alternative method.

7.  [HIN Support: HIN Mail to non-members](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): 30-day availability, SMS registration, trusted devices, supported browsers, support phone number.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Sending encrypted emails to recipients without HIN membership.

9.  [HIN Blog: The new HIN Client is here](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Browser sign-in, system requirements with TPM or Security Chip, rollout through October 26, 2026.

10.  [HIN Authenticator in the App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App for the second factor.
