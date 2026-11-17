---
title: "Proofpoint Essentials November 2026: neue Graph-Berechtigung für Message Pull bis Januar 2027"
navTitle: "Proofpoint Essentials 12.11.2026"
description: "Das Release von Proofpoint Essentials vom 12. November 2026 kündigt an, dass One-Click Message Pull und Automatic Remediation ab dem 1. Januar 2027 die Microsoft-Graph-Berechtigung Mail-Advanced.ReadWrite.All benötigen. Ergänzen Sie die Berechtigung in der Entra-App und erteilen Sie die Admin-Zustimmung vor dem Stichtag."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 Min. Lesezeit"
themen:
  - "proofpoint"
  - "microsoft-365-exchange"
produkte:
  - "proofpoint-essentials"
  - "proofpoint-releases"
  - "exchange-online"
  - "entra"
protokolle:
  - "releases"
  - "apis"
slug: "proofpoint-essentials-november-2026"
translationId: article-eb5da8564c33d2e9
url: https://rafaelpfister.ch/blog/proofpoint-essentials-november-2026
---

# Proofpoint Essentials November 2026: neue Graph-Berechtigung für Message Pull bis Januar 2027

Proofpoint hat am 12. November 2026 Release Notes für Proofpoint Essentials veröffentlicht, den Cloud-Dienst für E-Mail-Sicherheit in kleinen und mittleren Unternehmen. Der Eintrag ist das einzige Release im November 2026 und enthält keine neuen Funktionen oder Fehlerbehebungen: Er kündigt an, dass One-Click Message Pull und Automatic Remediation nach dem 1. Januar 2027 nur noch mit der zusätzlichen Microsoft-Graph-Berechtigung `Mail-Advanced.ReadWrite.All` funktionieren. Hintergrund ist eine Änderung von Microsoft, die ab dem 31. Dezember 2026 das Bearbeiten sensibler Eigenschaften versendeter oder empfangener Nachrichten über die Graph-API einschränkt. Der Hinweis richtet sich an Administratoren und Partner, die Proofpoint Essentials mit Microsoft 365 integriert haben und eine der beiden Funktionen nutzen. Betreiber des Proofpoint Protection Server (PPS, On-Premises) betrifft dieses Release nicht.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| Release vom 12. November 2026 | 12.11.2026 | Hinweis zu Berechtigungen | `Mail-Advanced.ReadWrite.All` für One-Click Message Pull und Automatic Remediation ab 1. Januar 2027 erforderlich |

## Microsoft-Graph-Berechtigung Mail-Advanced.ReadWrite.All

Laut Release Notes müssen Kunden, die One-Click Message Pull und Automatic Remediation über den 1. Januar 2027 hinaus nutzen wollen, ihre Microsoft-365-Integration um die Application Permission `Mail-Advanced.ReadWrite.All` (Microsoft Graph) erweitern. Die bestehenden Berechtigungen bleiben unverändert erhalten. In Proofpoint Essentials selbst ist keine weitere Konfiguration nötig.

Die Ursache liegt bei Microsoft. Im Microsoft 365 Developer Blog hat das Exchange-Team am 26. März 2026 angekündigt, dass Anwendungen sensible Eigenschaften von Nachrichten, die keine Entwürfe sind, ab dem 31. Dezember 2026 nur noch mit einer der Berechtigungen `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` oder `Mail-Advanced.ReadWrite.Shared` ändern dürfen. Als Beispiele nennt Microsoft Betreff, Nachrichtentext und Empfänger. Mit `Mail.ReadWrite` lassen sich danach nur noch Entwürfe in diesen Eigenschaften bearbeiten. Alle Mail-Advanced-Berechtigungen erfordern die Zustimmung eines Tenant-Administrators. Laut Microsoft stehen die Berechtigungen bereits zur Verfügung, die Ergänzung ist also vor dem Stichtag möglich.

Die Dokumentation zur Microsoft-365-Integration von Proofpoint Essentials führt für One-Click Message Pull bisher nur `Mail.ReadWrite` auf (Stand 17. November 2026). Mit dieser Berechtigung verschiebt Proofpoint Essentials Nachrichten direkt im Postfach des Benutzers, etwa in die oder aus der Quarantäne.

## Integration über die Microsoft Integration App

Wurde die Azure- bzw. Entra-Anwendung für Proofpoint Essentials automatisch über die Microsoft Integration App angelegt, ergänzen Sie die Berechtigung laut Release Notes so:

1. Melden Sie sich als Administrator mit Zugriff auf den betreffenden Kunden bei Proofpoint Essentials an (Partner-Admin, Channel-Admin oder Org-Admin).
2. Öffnen Sie **Account Management > Integrations**.
3. Klicken Sie auf der Kachel **Microsoft 365** auf **Details**.
4. Klicken Sie auf **Update Permissions**.
5. Lassen Sie die anschliessende Consent-Abfrage von einem Microsoft-365-Administrator mit ausreichender Rolle bestätigen, zum Beispiel Global Administrator oder Privileged Role Administrator.

Danach ergänzt die App die Berechtigung `Mail-Advanced.ReadWrite.All`.

## Manuell angelegte Entra-Anwendung

Wurde die Anwendung manuell registriert, gilt laut Release Notes dieses Vorgehen:

1. Melden Sie sich im Microsoft Entra Admin Center (`https://entra.microsoft.com`) mit einem Konto an, das Admin-Zustimmungen erteilen darf (zum Beispiel Global Administrator oder Privileged Role Administrator).
2. Öffnen Sie **Identity > Applications > App registrations**.
3. Öffnen Sie die App-Registrierung für Proofpoint Essentials. Erscheint sie nicht, wechseln Sie auf den Reiter **All applications**.
4. Wählen Sie **API permissions** und dann **Add a permission > Microsoft Graph > Application permissions**.
5. Suchen Sie `Mail-Advanced.ReadWrite.All`, markieren Sie die Berechtigung und wählen Sie **Add permissions**.
6. Wählen Sie **Grant admin consent for <Organisation>** und bestätigen Sie.
7. Prüfen Sie, dass beim Status der Berechtigung **Granted for <Organisation>** steht.

Alle bestehenden Berechtigungen bleiben laut Proofpoint erhalten.

## Empfohlenes Vorgehen

Ergänzen Sie `Mail-Advanced.ReadWrite.All` vor dem 31. Dezember 2026, dem Datum, ab dem Microsoft die Einschränkung durchsetzt; Proofpoint nennt als Stichtag den 1. Januar 2027. Klären Sie zuerst, ob die Entra-Anwendung über die Microsoft Integration App oder manuell angelegt wurde, und folgen Sie dem passenden Ablauf. Halten Sie dafür einen Administrator mit der Rolle Global Administrator oder Privileged Role Administrator bereit. Prüfen Sie nach der Zustimmung im Entra Admin Center, dass die Berechtigung als erteilt angezeigt wird. Was ohne die neue Berechtigung nach dem Stichtag im Detail geschieht, beschreiben die Release Notes nicht; sie nennen nur, dass die Berechtigung für die weitere Nutzung der beiden Funktionen erforderlich ist (Stand 17. November 2026).

## Quellen

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): Eintrag vom 12. November 2026 mit der neuen Berechtigung, dem Stichtag 1. Januar 2027 und den Schritten für beide Integrationsarten.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): Bisherige Graph-Berechtigungen der Integration, darunter `Mail.ReadWrite` für One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): Ankündigung vom 26. März 2026, Durchsetzung ab 31. Dezember 2026, Mail-Advanced-Berechtigungen mit Admin-Zustimmung.
