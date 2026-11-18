---
title: "Proofpoint Essentials November 2026: New Graph Permission for Message Pull Required by January 2027"
navTitle: "Proofpoint Essentials 11/12/2026"
description: "The Proofpoint Essentials release of November 12, 2026, announces that One-Click Message Pull and Automatic Remediation will require the Microsoft Graph permission Mail-Advanced.ReadWrite.All as of January 1, 2027. Add the permission to the Entra app and grant admin consent before the deadline."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min read"
themen:
  - proofpoint
  - microsoft-365-exchange
produkte:
  - "proofpoint-essentials"
  - "proofpoint-releases"
  - "exchange-online"
  - "entra"
protokolle:
  - "releases"
  - "apis"
slug: "proofpoint-essentials-november-2026-new-graph-permission-for-message-pull-required-by-january"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/en/blog/proofpoint-essentials-november-2026-new-graph-permission-for-message-pull-required-by-january
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-17T21:10:56.414Z
translationReview: required
---

# Proofpoint Essentials November 2026: New Graph Permission for Message Pull Required by January 2027

On November 12, 2026, Proofpoint published release notes for Proofpoint Essentials, the cloud service for email security in small and medium-sized businesses. The entry is the only release in November 2026 and contains no new features or bug fixes: it announces that One-Click Message Pull and Automatic Remediation will only work after January 1, 2027, with the additional Microsoft Graph permission `Mail-Advanced.ReadWrite.All`. The reason is a Microsoft change that, starting December 31, 2026, restricts the editing of sensitive properties of sent or received messages through the Graph API. The notice is aimed at administrators and partners who have integrated Proofpoint Essentials with Microsoft 365 and use one of the two features. This release does not affect operators of Proofpoint Protection Server (PPS, on-premises).

## Version overview

| Version | Date | Type | Key points |
|---|---|---|---|
| Release of November 12, 2026 | 11/12/2026 | Permission notice | `Mail-Advanced.ReadWrite.All` required for One-Click Message Pull and Automatic Remediation starting January 1, 2027 |

## Microsoft Graph permission Mail-Advanced.ReadWrite.All

According to the release notes, customers who want to continue using One-Click Message Pull and Automatic Remediation beyond January 1, 2027, must extend their Microsoft 365 integration with the application permission `Mail-Advanced.ReadWrite.All` (Microsoft Graph). Existing permissions remain unchanged. No further configuration is required in Proofpoint Essentials itself.

The cause lies with Microsoft. In the Microsoft 365 Developer Blog, the Exchange team announced on March 26, 2026, that starting December 31, 2026, applications may only modify sensitive properties of messages that are not drafts with one of the permissions `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All`, or `Mail-Advanced.ReadWrite.Shared`. Microsoft cites the subject, message body, and recipients as examples. With `Mail.ReadWrite`, only drafts can subsequently be edited in these properties. All Mail-Advanced permissions require consent from a tenant administrator. According to Microsoft, the permissions are already available, so they can be added before the deadline.

The Proofpoint Essentials documentation for the Microsoft 365 integration currently lists only `Mail.ReadWrite` for One-Click Message Pull (as of November 17, 2026). With this permission, Proofpoint Essentials moves messages directly in the user's mailbox, for example, to or from quarantine.

## Integration through the Microsoft Integration App

If the Azure or Entra application for Proofpoint Essentials was created automatically through the Microsoft Integration App, add the permission as follows according to the release notes:

1. Sign in to Proofpoint Essentials as an administrator with access to the relevant customer (Partner Admin, Channel Admin, or Org Admin).
2. Open **Account Management > Integrations**.
3. On the **Microsoft 365** tile, click **Details**.
4. Click **Update Permissions**.
5. Have the subsequent consent prompt confirmed by a Microsoft 365 administrator with a sufficient role, such as Global Administrator or Privileged Role Administrator.

The app then adds the permission `Mail-Advanced.ReadWrite.All`.

## Manually created Entra application

If the application was registered manually, the release notes specify the following procedure:

1. Sign in to the Microsoft Entra admin center (`https://entra.microsoft.com`) with an account authorized to grant admin consent (for example, Global Administrator or Privileged Role Administrator).
2. Open **Identity > Applications > App registrations**.
3. Open the app registration for Proofpoint Essentials. If it does not appear, switch to the **All applications** tab.
4. Select **API permissions**, then **Add a permission > Microsoft Graph > Application permissions**.
5. Search for `Mail-Advanced.ReadWrite.All`, select the permission, and choose **Add permissions**.
6. Select **Grant admin consent for <Organisation>** and confirm.
7. Verify that the permission status says **Granted for <Organisation>**.

According to Proofpoint, all existing permissions are retained.

## Recommended approach

Add `Mail-Advanced.ReadWrite.All` before December 31, 2026, the date from which Microsoft enforces the restriction; Proofpoint gives January 1, 2027, as the deadline. First determine whether the Entra application was created through the Microsoft Integration App or manually, then follow the appropriate procedure. Have an administrator with the Global Administrator or Privileged Role Administrator role available. After consent, verify in the Entra admin center that the permission is shown as granted. The release notes do not describe in detail what happens after the deadline without the new permission; they only state that the permission is required for continued use of the two features (as of November 17, 2026).

## Sources

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): Entry of November 12, 2026, with the new permission, the January 1, 2027 deadline, and the steps for both integration types.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): Existing Graph permissions for the integration, including `Mail.ReadWrite` for One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): Announcement of March 26, 2026, enforcement starting December 31, 2026, and Mail-Advanced permissions with admin consent.
