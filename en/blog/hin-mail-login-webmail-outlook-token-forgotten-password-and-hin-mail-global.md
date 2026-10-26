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
translationSourceHash: 00c1c2a195ebd80ccf8c6f7e7318d3b924eff80efe6315bd1d6fc95c05ad3c9f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:29:10.200Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/hin-mail-login-webmail-outlook-token-forgotten-password-and-hin-mail-global
---

# HIN Mail Login: Webmail, Outlook Token, Forgotten Password, and HIN Mail Global

There are usually three different situations behind a search for “HIN Mail Login”: You are a HIN member and want to read your email in a browser, you want to set up HIN Mail in Outlook or on your smartphone, or you are a patient or external organization that has received an encrypted HIN email and do not have a HIN account at all. The sign-in process differs in each case.

| Situation | Address | Username | Password |
|---|---|---|---|
| Webmail in the browser | `webmail.hin.ch` | HIN ID | HIN password, secured by HIN Client, SMS code, or Authenticator app |
| Outlook, Thunderbird, Apple Mail, smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | Mail token from `apps.hin.ch` |
| Administration, token, initialization password | `apps.hin.ch` | HIN ID | HIN password plus second factor |
| Recipient without a HIN account (HIN Mail Global) | Attachment `secure-email.html` or link in the email | Mobile number | SMS code |

## HIN Webmail: signing in through the browser

You can access webmail at `https://webmail.hin.ch`. Sign in with your HIN ID (your HIN username, e.g., `hmuster`) and HIN password. The second factor is provided by one of the following methods:

- **HIN Client:** If the client is running on the workstation and you are signed in, webmail opens without any further prompt. With HIN Client 4.0, starting in early November 2026, you will sign in directly in the browser: enter the HIN-protected address and you will be redirected to the sign-in page.
- **SMS code (mTAN):** After entering your password, you receive a code at the stored mobile number. This method is suitable for devices without HIN Client, such as a personal laptop.
- **HIN Authenticator App:** The app (App Store and Google Play, “HIN Authenticator”) confirms the sign-in on your smartphone.

The SMS code and Authenticator app must be activated in advance. If neither is available and HIN Client is unavailable, you cannot sign in to webmail. Therefore, set up at least one alternative while the client is still working.

In webmail, you can compose and manage email, maintain contacts and address books, search the HIN participant directory, and create multiple signatures. A bar below the username shows the available mailbox storage; if it is red, you must delete emails or expand the storage.

## HIN Mail in Outlook and on your smartphone: signing in with a mail token

Email clients do not sign in with the HIN password, but with a **mail token**. This is the most common reason Outlook repeatedly asks for a password: the HIN password is rejected during IMAP login.

To create the token:

1. Open `https://apps.hin.ch` and sign in with your HIN ID (HIN Client, SMS code, or Authenticator app).
2. Select the “HIN Mail” section.
3. Click “Add device” and select “Mail Clients” as the device type.
4. Copy the displayed token and enter it as the password in the email client.

The server settings:

| Setting | Incoming mail (IMAP) | Outgoing mail (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Encryption | SSL/TLS | STARTTLS |
| Username | HIN ID | HIN ID |
| Password | Mail token | Mail token |
| Authentication | standard | Enable “Outgoing server requires authentication” |

The username is the HIN ID, not the email address. Outlook suggests the email address during automatic setup; you must therefore set it up manually using “Advanced options” or “Manual setup.”

The token is tied to your HIN identity and can be revoked and recreated at any time in `apps.hin.ch`. A separate token is recommended for every device: if a smartphone is lost, you only disable that device, while Outlook on the practice PC continues to work. Once the token is configured, Outlook no longer needs HIN Client to retrieve email.

## First sign-in without HIN Client: activate your identity

New HIN identities can be activated without HIN Client. You need the username and initialization password from your HIN documents.

1. Open the activation link in the Service Center (`servicecenter.hin.ch`).
2. Enter your username and initialization password.
3. Set your own password: at least 10 characters, including numbers, uppercase and lowercase letters, and special characters.
4. Set up a second factor immediately (SMS code or Authenticator app).
5. Then sign in at `apps.hin.ch`.

You have 10 minutes for activation. After that, the initialization password is reset for security reasons, and you must request a new one.

## Forgotten HIN password

The reset depends on whether an alternative sign-in method has been configured.

**With an SMS code or Authenticator app:**

1. Sign in at `apps.hin.ch` using the alternative method.
2. Select the “HIN Client” tab and create a new initialization password.
3. In HIN Client, select “Registration,” enter the username and new initialization password.
4. Set a new password and confirm “Register new HIN identity.”

**Without an alternative method and without a signed-in device,** the only option is HIN Support by phone (0848 830 740, Monday through Friday, 8:00 a.m. to 6:00 p.m.). Write down the new initialization password before closing the window.

## HIN Mail Global: open encrypted email without a HIN account

Medical practices, hospitals, and laboratories send messages to patients or organizations without HIN membership through HIN Mail Global. The email arrives as a regular email, but its contents are encrypted in attachment `secure-email.html` or behind the “Read encrypted message” link. There is no HIN login in the usual sense; you sign in using your mobile number.

**The first time:**

1. Download and open attachment `secure-email.html` (or click the link in the email). With webmail services such as Gmail, Bluewin, or GMX, save it first, then open it.
2. Select a language and enter your mobile number.
3. Enter the code received by SMS. The message is displayed.

**For subsequent messages,** simply open the attachment; the SMS code is automatically sent to the registered number.

Please note:

- The message is available through the link for **30 days**, after which access expires. Save or print important contents within this period.
- Only the original recipient can open forwarded messages.
- You can reply in encrypted form.
- A device can be marked as trusted; the SMS prompt is then waived for one year.
- If no code arrives within 15 minutes, you can request a new one.
- The two current versions of Chrome, Safari, Edge, and Firefox are supported.

If you have problems with HIN Mail Global, HIN Mail Support can help at +41 58 670 48 69. Only the sender can answer questions about the message content.

## Common HIN Mail login errors

| Symptom | Cause | Solution |
|---|---|---|
| Outlook repeatedly asks for the password | HIN password entered instead of mail token | Create a token in `apps.hin.ch` and enter it as the password |
| Outlook sign-in fails despite the token | Email address used as the username | Use the HIN ID as the username |
| Sending fails, receiving works | SMTP authentication not enabled, incorrect port, or incorrect encryption | Use port `587` with STARTTLS and enable authentication |
| Webmail requests a code that never arrives | Mobile number is outdated or SMS code is not activated | Use the Authenticator app, or sign in through HIN Client and update the number |
| Initialization password is rejected | The 10-minute time limit has expired | Request a new initialization password |
| Token worked but suddenly no longer does | Token revoked in `apps.hin.ch` or device removed | Create a new token for this device |
| HIN Mail Global: link no longer opens | 30-day period has expired | Ask the sender to resend it |

## HIN Client 4.0: what changes with sign-in

HIN will automatically distribute HIN Client 4.0 by October 26, 2026; anyone who does not receive an automatic update can download it from `download.hin.ch`. The existing password remains valid; when you first sign in with the new version, you will be asked to change it once. Starting in early November 2026, you will sign in to HIN-protected services such as webmail directly in the browser.

Version 4.0 requires Windows 11 with TPM or macOS 14 with a Security Chip. Devices that do not meet these requirements can no longer use the client. Access to webmail remains possible on those devices using an SMS code or Authenticator app, and email retrieval in Outlook remains possible using the mail token. Details about the deadlines for the platform renewal are in the article [HIN Platform Renewal 2026](/blog/hin-plattformerneuerung-2026).

## Medical practices and hospitals with their own mail server

Organizations with their own HIN Mailgateway do not sign in to HIN Mail. The mailboxes are hosted on their own Exchange or in Exchange Online, and the gateway (a SEPPmail appliance) encrypts email on its way to and from the HIN community. Login issues there concern the organization's own mail system, not the HIN platform. This gateway is being replaced by the new HIN Gateway (“Stargate”).

<aside class="offer-box">
  <span class="offer-box__tag">Free assessment</span>
  <p><strong>Do you operate your own HIN Mailgateway?</strong> I will review your environment and tell you what needs to be done before migration to “Stargate.”</p>
  <a class="offer-box__cta" href="/stargate">Register now</a>
</aside>

## Change log

- **October 25, 2026:** Corrected: When first signing in with HIN Client 4.0, the password must be changed once, and browser sign-in does not apply until early November 2026.

## Sources

1.  [HIN Support: HIN Mail and Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Overview with the webmail address and instructions for Outlook, Apple Mail, Thunderbird, iOS, and Android.

2.  [HIN Support: How does webmail work?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Webmail functions, storage display.

3.  [HIN Support: Mail Token Service guide for Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Token creation in apps.hin.ch, IMAP and SMTP servers, ports, and encryption.

4.  [HIN Support: What is a token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): Mail tokens and access tokens, tied to the HIN identity.

5.  [HIN Support: Activate HIN identity (without HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Activation in the Service Center, 10-minute time limit, password rules, second factor.

6.  [HIN Support: Forgot password](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Reset using an initialization password, phone support without an alternative method.

7.  [HIN Support: HIN Mail to non-members](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): Available for 30 days, SMS registration, trusted devices, supported browsers, support phone number.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Sending encrypted email to recipients without HIN membership.

9.  [HIN Blog: The new HIN Client is here](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Browser sign-in starting in early November 2026, password change at first sign-in, system requirements with TPM or Security Chip, rollout by October 26, 2026.

10.  [HIN Authenticator in the App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App for the second factor.
