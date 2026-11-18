---
title: "Proofpoint Essentials november 2026: ny Graph-behörighet för Message Pull till januari 2027"
navTitle: "Proofpoint Essentials 2026-11-12"
description: "Utgåvan av Proofpoint Essentials den 12 november 2026 meddelar att One-Click Message Pull och Automatic Remediation från och med den 1 januari 2027 kräver Microsoft Graph-behörigheten Mail-Advanced.ReadWrite.All. Lägg till behörigheten i Entra-appen och ge administratörsmedgivande före tidsfristen."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min lästid"
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
slug: "proofpoint-essentials-november-2026-ny-graph-behorighet-for-message-pull-till-januari-2027"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/sv/blog/proofpoint-essentials-november-2026-ny-graph-behorighet-for-message-pull-till-januari-2027
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:24:49.179Z
translationReview: required
---

# Proofpoint Essentials november 2026: ny Graph-behörighet för Message Pull till januari 2027

Proofpoint publicerade den 12 november 2026 versionsinformation för Proofpoint Essentials, molntjänsten för e-postsäkerhet i små och medelstora företag. Inlägget är den enda utgåvan i november 2026 och innehåller inga nya funktioner eller felrättningar: det meddelar att One-Click Message Pull och Automatic Remediation efter den 1 januari 2027 endast fungerar med den extra Microsoft Graph-behörigheten `Mail-Advanced.ReadWrite.All`. Bakgrunden är en ändring från Microsoft som från och med den 31 december 2026 begränsar redigering av känsliga egenskaper för skickade eller mottagna meddelanden via Graph API. Informationen riktar sig till administratörer och partner som har integrerat Proofpoint Essentials med Microsoft 365 och använder någon av de två funktionerna. Operatörer av Proofpoint Protection Server (PPS, lokalt installerad) berörs inte av denna utgåva.

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| Utgåva från 12 november 2026 | 2026-11-12 | Behörighetsinformation | `Mail-Advanced.ReadWrite.All` krävs för One-Click Message Pull och Automatic Remediation från 1 januari 2027 |

## Microsoft Graph-behörigheten Mail-Advanced.ReadWrite.All

Enligt versionsinformationen måste kunder som vill använda One-Click Message Pull och Automatic Remediation även efter den 1 januari 2027 utöka sin Microsoft 365-integration med programbehörigheten `Mail-Advanced.ReadWrite.All` (Microsoft Graph). Befintliga behörigheter kvarstår oförändrade. Ingen ytterligare konfiguration krävs i själva Proofpoint Essentials.

Orsaken ligger hos Microsoft. I Microsoft 365 Developer Blog meddelade Exchange-teamet den 26 mars 2026 att program från och med den 31 december 2026 endast får ändra känsliga egenskaper för meddelanden som inte är utkast med någon av behörigheterna `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` eller `Mail-Advanced.ReadWrite.Shared`. Microsoft nämner ämne, meddelandetext och mottagare som exempel. Med `Mail.ReadWrite` går det därefter endast att redigera dessa egenskaper i utkast. Alla Mail-Advanced-behörigheter kräver godkännande från en klientadministratör. Enligt Microsoft finns behörigheterna redan tillgängliga, så tillägget kan göras före tidsfristen.

Dokumentationen för Microsoft 365-integrationen i Proofpoint Essentials listar hittills endast `Mail.ReadWrite` för One-Click Message Pull (status 17 november 2026). Med denna behörighet flyttar Proofpoint Essentials meddelanden direkt i användarens postlåda, exempelvis till eller från karantänen.

## Integration via Microsoft Integration App

Om Azure- eller Entra-programmet för Proofpoint Essentials skapades automatiskt via Microsoft Integration App lägger du enligt versionsinformationen till behörigheten så här:

1. Logga in i Proofpoint Essentials som administratör med åtkomst till den berörda kunden (Partner Admin, Channel Admin eller Org Admin).
2. Öppna **Account Management > Integrations**.
3. Klicka på **Details** på panelen **Microsoft 365**.
4. Klicka på **Update Permissions**.
5. Låt den efterföljande samtyckesbegäran bekräftas av en Microsoft 365-administratör med tillräcklig roll, till exempel Global Administrator eller Privileged Role Administrator.

Därefter lägger appen till behörigheten `Mail-Advanced.ReadWrite.All`.

## Manuellt skapat Entra-program

Om programmet har registrerats manuellt gäller följande procedur enligt versionsinformationen:

1. Logga in i Microsoft Entra Admin Center (`https://entra.microsoft.com`) med ett konto som får ge administratörsmedgivanden (till exempel Global Administrator eller Privileged Role Administrator).
2. Öppna **Identity > Applications > App registrations**.
3. Öppna appregistreringen för Proofpoint Essentials. Om den inte visas växlar du till fliken **All applications**.
4. Välj **API permissions** och sedan **Add a permission > Microsoft Graph > Application permissions**.
5. Sök efter `Mail-Advanced.ReadWrite.All`, markera behörigheten och välj **Add permissions**.
6. Välj **Grant admin consent for <Organisation>** och bekräfta.
7. Kontrollera att behörighetens status är **Granted for <Organisation>**.

Enligt Proofpoint kvarstår alla befintliga behörigheter.

## Rekommenderad procedur

Lägg till `Mail-Advanced.ReadWrite.All` före den 31 december 2026, då Microsoft börjar tillämpa begränsningen; Proofpoint anger den 1 januari 2027 som tidsfrist. Klargör först om Entra-programmet skapades via Microsoft Integration App eller manuellt och följ rätt process. Ha en administratör med rollen Global Administrator eller Privileged Role Administrator tillgänglig för detta. Kontrollera efter medgivandet i Entra Admin Center att behörigheten visas som beviljad. Versionsinformationen beskriver inte i detalj vad som händer efter tidsfristen utan den nya behörigheten; den anger endast att behörigheten krävs för fortsatt användning av de två funktionerna (status 17 november 2026).

## Källor

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): inlägg från 12 november 2026 med den nya behörigheten, tidsfristen 1 januari 2027 och stegen för båda integrationstyperna.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): befintliga Graph-behörigheter för integrationen, däribland `Mail.ReadWrite` för One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): meddelande från 26 mars 2026, tillämpning från 31 december 2026, Mail-Advanced-behörigheter med administratörsmedgivande.
