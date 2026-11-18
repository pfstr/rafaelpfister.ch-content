---
title: "Proofpoint Essentials november 2026: Ny Graph-tillatelse for Message Pull innen januar 2027"
navTitle: "Proofpoint Essentials 12.11.2026"
description: "Utgivelsen av Proofpoint Essentials fra 12. november 2026 varsler at One-Click Message Pull og Automatic Remediation fra 1. januar 2027 trenger Microsoft Graph-tillatelsen Mail-Advanced.ReadWrite.All. Legg til tillatelsen i Entra-appen og gi administratorsamtykke før fristen."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min lesetid"
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
slug: "proofpoint-essentials-november-2026-ny-graph-tillatelse-for-message-pull-innen-januar-2027"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/no/blog/proofpoint-essentials-november-2026-ny-graph-tillatelse-for-message-pull-innen-januar-2027
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:25:03.526Z
translationReview: automatic
---

# Proofpoint Essentials november 2026: Ny Graph-tillatelse for Message Pull innen januar 2027

Proofpoint publiserte 12. november 2026 utgivelsesnotater for Proofpoint Essentials, skytjenesten for e-postsikkerhet i små og mellomstore bedrifter. Oppføringen er den eneste utgivelsen i november 2026 og inneholder ingen nye funksjoner eller feilrettinger: Den varsler at One-Click Message Pull og Automatic Remediation etter 1. januar 2027 bare vil fungere med den ekstra Microsoft Graph-tillatelsen `Mail-Advanced.ReadWrite.All`. Bakgrunnen er en endring fra Microsoft som fra 31. desember 2026 begrenser redigering av sensitive egenskaper i sendte eller mottatte meldinger via Graph-API-et. Merknaden er rettet mot administratorer og partnere som har integrert Proofpoint Essentials med Microsoft 365 og bruker én av de to funksjonene. Operatører av Proofpoint Protection Server (PPS, lokalt installert) påvirkes ikke av denne utgivelsen.

## Versjonene i oversikt

| Versjon | Dato | Type | Hovedpunkter |
|---|---|---|---|
| Utgivelse fra 12. november 2026 | 12.11.2026 | Merknad om tillatelser | `Mail-Advanced.ReadWrite.All` kreves for One-Click Message Pull og Automatic Remediation fra 1. januar 2027 |

## Microsoft Graph-tillatelsen Mail-Advanced.ReadWrite.All

Ifølge utgivelsesnotatene må kunder som vil bruke One-Click Message Pull og Automatic Remediation etter 1. januar 2027, utvide Microsoft 365-integrasjonen med applikasjonstillatelsen `Mail-Advanced.ReadWrite.All` (Microsoft Graph). De eksisterende tillatelsene forblir uendret. Det er ikke nødvendig med ytterligere konfigurasjon i selve Proofpoint Essentials.

Årsaken ligger hos Microsoft. I Microsoft 365 Developer Blog annonserte Exchange-teamet 26. mars 2026 at applikasjoner fra 31. desember 2026 bare kan endre sensitive egenskaper i meldinger som ikke er utkast, med én av tillatelsene `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` eller `Mail-Advanced.ReadWrite.Shared`. Microsoft nevner emne, meldingstekst og mottakere som eksempler. Med `Mail.ReadWrite` kan bare utkast fortsatt redigeres i disse egenskapene. Alle Mail-Advanced-tillatelser krever samtykke fra en tenant-administrator. Ifølge Microsoft er tillatelsene allerede tilgjengelige, slik at de kan legges til før fristen.

Dokumentasjonen for Microsoft 365-integrasjonen i Proofpoint Essentials oppgir foreløpig bare `Mail.ReadWrite` for One-Click Message Pull (per 17. november 2026). Med denne tillatelsen flytter Proofpoint Essentials meldinger direkte i brukerens postboks, for eksempel til eller fra karantenen.

## Integrasjon via Microsoft Integration App

Hvis Azure- eller Entra-applikasjonen for Proofpoint Essentials ble opprettet automatisk via Microsoft Integration App, legger du til tillatelsen slik i henhold til utgivelsesnotatene:

1. Logg på Proofpoint Essentials som administrator med tilgang til den aktuelle kunden (Partner-Admin, Channel-Admin eller Org-Admin).
2. Åpne **Account Management > Integrations**.
3. Klikk på **Details** på flisen **Microsoft 365**.
4. Klikk på **Update Permissions**.
5. La den påfølgende samtykkeforespørselen bli bekreftet av en Microsoft 365-administrator med tilstrekkelig rolle, for eksempel Global Administrator eller Privileged Role Administrator.

Deretter legger appen til tillatelsen `Mail-Advanced.ReadWrite.All`.

## Manuelt opprettet Entra-applikasjon

Hvis applikasjonen ble registrert manuelt, gjelder denne fremgangsmåten ifølge utgivelsesnotatene:

1. Logg på Microsoft Entra Admin Center (`https://entra.microsoft.com`) med en konto som kan gi administratorsamtykke (for eksempel Global Administrator eller Privileged Role Administrator).
2. Åpne **Identity > Applications > App registrations**.
3. Åpne appregistreringen for Proofpoint Essentials. Hvis den ikke vises, bytter du til fanen **All applications**.
4. Velg **API permissions** og deretter **Add a permission > Microsoft Graph > Application permissions**.
5. Søk etter `Mail-Advanced.ReadWrite.All`, merk tillatelsen og velg **Add permissions**.
6. Velg **Grant admin consent for <Organisation>** og bekreft.
7. Kontroller at tillatelsesstatusen viser **Granted for <Organisation>**.

Alle eksisterende tillatelser forblir ifølge Proofpoint bevart.

## Anbefalt fremgangsmåte

Legg til `Mail-Advanced.ReadWrite.All` før 31. desember 2026, datoen da Microsoft håndhever begrensningen; Proofpoint oppgir 1. januar 2027 som frist. Avklar først om Entra-applikasjonen ble opprettet via Microsoft Integration App eller manuelt, og følg den passende prosessen. Sørg for at en administrator med rollen Global Administrator eller Privileged Role Administrator er tilgjengelig. Kontroller etter samtykket i Entra Admin Center at tillatelsen vises som gitt. Utgivelsesnotatene beskriver ikke nøyaktig hva som skjer uten den nye tillatelsen etter fristen; de oppgir bare at tillatelsen er nødvendig for fortsatt bruk av de to funksjonene (per 17. november 2026).

## Kilder

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): Oppføring fra 12. november 2026 med den nye tillatelsen, fristen 1. januar 2027 og trinnene for begge integrasjonstypene.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): Tidligere Graph-tillatelser for integrasjonen, inkludert `Mail.ReadWrite` for One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): Kunngjøring fra 26. mars 2026, håndheving fra 31. desember 2026, Mail-Advanced-tillatelser med administratorsamtykke.
