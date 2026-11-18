---
title: "Proofpoint Essentials novembre 2026: nuova autorizzazione Graph per Message Pull entro gennaio 2027"
navTitle: "Proofpoint Essentials 12.11.2026"
description: "La release di Proofpoint Essentials del 12 novembre 2026 annuncia che One-Click Message Pull e Automatic Remediation richiederanno l'autorizzazione Microsoft Graph Mail-Advanced.ReadWrite.All a partire dal 1° gennaio 2027. Aggiungere l'autorizzazione nell'app Entra e concedere il consenso dell'amministratore prima della scadenza."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min di lettura"
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
slug: "proofpoint-essentials-novembre-2026-nuova-autorizzazione-graph-per-message-pull-entro-gennaio"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/it/blog/proofpoint-essentials-novembre-2026-nuova-autorizzazione-graph-per-message-pull-entro-gennaio
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:24:21.608Z
translationReview: automatic
---

# Proofpoint Essentials novembre 2026: nuova autorizzazione Graph per Message Pull entro gennaio 2027

Il 12 novembre 2026 Proofpoint ha pubblicato le note di rilascio di Proofpoint Essentials, il servizio cloud per la sicurezza delle e-mail nelle piccole e medie imprese. La voce è l'unica release di novembre 2026 e non contiene nuove funzionalità né correzioni di errori: annuncia che One-Click Message Pull e Automatic Remediation funzioneranno dopo il 1° gennaio 2027 solo con l'autorizzazione aggiuntiva di Microsoft Graph `Mail-Advanced.ReadWrite.All`. Il motivo è una modifica di Microsoft che, a partire dal 31 dicembre 2026, limita tramite l'API Graph la modifica delle proprietà sensibili dei messaggi inviati o ricevuti. L'avviso è rivolto ad amministratori e partner che hanno integrato Proofpoint Essentials con Microsoft 365 e utilizzano una delle due funzionalità. Questa release non riguarda gli operatori di Proofpoint Protection Server (PPS, on-premises).

## Le versioni in sintesi

| Versione | Data | Tipo | Punti chiave |
|---|---|---|---|
| Release del 12 novembre 2026 | 12.11.2026 | Avviso sulle autorizzazioni | `Mail-Advanced.ReadWrite.All` richiesto per One-Click Message Pull e Automatic Remediation dal 1° gennaio 2027 |

## Autorizzazione Microsoft Graph Mail-Advanced.ReadWrite.All

Secondo le note di rilascio, i clienti che desiderano continuare a utilizzare One-Click Message Pull e Automatic Remediation oltre il 1° gennaio 2027 devono estendere la loro integrazione Microsoft 365 con l'autorizzazione applicativa `Mail-Advanced.ReadWrite.All` (Microsoft Graph). Le autorizzazioni esistenti rimangono invariate. Non è necessaria alcuna ulteriore configurazione in Proofpoint Essentials.

La causa risiede in Microsoft. Nel Microsoft 365 Developer Blog, il team Exchange ha annunciato il 26 marzo 2026 che, dal 31 dicembre 2026, le applicazioni potranno modificare le proprietà sensibili dei messaggi che non sono bozze solo con una delle autorizzazioni `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` o `Mail-Advanced.ReadWrite.Shared`. Microsoft cita come esempi l'oggetto, il corpo del messaggio e i destinatari. Con `Mail.ReadWrite` sarà poi possibile modificare tali proprietà solo nelle bozze. Tutte le autorizzazioni Mail-Advanced richiedono il consenso di un amministratore del tenant. Secondo Microsoft, le autorizzazioni sono già disponibili, quindi possono essere aggiunte prima della scadenza.

La documentazione relativa all'integrazione Microsoft 365 di Proofpoint Essentials elenca finora per One-Click Message Pull solo `Mail.ReadWrite` (al 17 novembre 2026). Con questa autorizzazione, Proofpoint Essentials sposta i messaggi direttamente nella casella di posta dell'utente, ad esempio verso o dalla quarantena.

## Integrazione tramite Microsoft Integration App

Se l'applicazione Azure o Entra per Proofpoint Essentials è stata creata automaticamente tramite Microsoft Integration App, secondo le note di rilascio aggiungere l'autorizzazione come segue:

1. Accedere a Proofpoint Essentials come amministratore con accesso al cliente interessato (Partner Admin, Channel Admin o Org Admin).
2. Aprire **Account Management > Integrations**.
3. Nella scheda **Microsoft 365**, fare clic su **Details**.
4. Fare clic su **Update Permissions**.
5. Fare confermare la successiva richiesta di consenso da un amministratore Microsoft 365 con un ruolo sufficiente, ad esempio Global Administrator o Privileged Role Administrator.

L'app aggiungerà quindi l'autorizzazione `Mail-Advanced.ReadWrite.All`.

## Applicazione Entra creata manualmente

Se l'applicazione è stata registrata manualmente, secondo le note di rilascio si applica la seguente procedura:

1. Accedere al Microsoft Entra Admin Center (`https://entra.microsoft.com`) con un account autorizzato a concedere consensi amministrativi (ad esempio Global Administrator o Privileged Role Administrator).
2. Aprire **Identity > Applications > App registrations**.
3. Aprire la registrazione dell'app per Proofpoint Essentials. Se non appare, passare alla scheda **All applications**.
4. Selezionare **API permissions**, quindi **Add a permission > Microsoft Graph > Application permissions**.
5. Cercare `Mail-Advanced.ReadWrite.All`, selezionare l'autorizzazione e scegliere **Add permissions**.
6. Selezionare **Grant admin consent for <Organisation>** e confermare.
7. Verificare che lo stato dell'autorizzazione riporti **Granted for <Organisation>**.

Secondo Proofpoint, tutte le autorizzazioni esistenti rimangono invariate.

## Procedura consigliata

Aggiungere `Mail-Advanced.ReadWrite.All` prima del 31 dicembre 2026, la data a partire dalla quale Microsoft applicherà la limitazione; Proofpoint indica il 1° gennaio 2027 come scadenza. Verificare innanzitutto se l'applicazione Entra è stata creata tramite Microsoft Integration App o manualmente, quindi seguire la procedura appropriata. Assicurarsi che sia disponibile un amministratore con il ruolo Global Administrator o Privileged Role Administrator. Dopo il consenso, verificare nell'Entra Admin Center che l'autorizzazione sia indicata come concessa. Le note di rilascio non descrivono nel dettaglio cosa accada dopo la scadenza senza la nuova autorizzazione; indicano soltanto che l'autorizzazione è necessaria per continuare a utilizzare le due funzionalità (al 17 novembre 2026).

## Fonti

1.  [Proofpoint Essentials: Note di rilascio](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): voce del 12 novembre 2026 con la nuova autorizzazione, la scadenza del 1° gennaio 2027 e i passaggi per entrambi i tipi di integrazione.

2.  [Proofpoint Essentials: Integrazione MX di Microsoft 365](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): autorizzazioni Graph dell'integrazione finora previste, inclusa `Mail.ReadWrite` per One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: aggiornamenti dell'API Graph alle proprietà sensibili delle e-mail](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): annuncio del 26 marzo 2026, applicazione dal 31 dicembre 2026, autorizzazioni Mail-Advanced con consenso dell'amministratore.
