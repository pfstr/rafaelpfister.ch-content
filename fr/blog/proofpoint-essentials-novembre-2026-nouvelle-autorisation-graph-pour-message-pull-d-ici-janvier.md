---
title: "Proofpoint Essentials novembre 2026 : nouvelle autorisation Graph pour Message Pull d’ici janvier 2027"
navTitle: "Proofpoint Essentials 12.11.2026"
description: "La version de Proofpoint Essentials du 12 novembre 2026 annonce que One-Click Message Pull et Automatic Remediation nécessiteront l’autorisation Microsoft Graph Mail-Advanced.ReadWrite.All à partir du 1er janvier 2027. Ajoutez l’autorisation dans l’application Entra et accordez le consentement administrateur avant la date limite."
date: "2026-11-17"
kategorie: "Proofpoint"
timeToRead: "4 min de lecture"
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
slug: "proofpoint-essentials-novembre-2026-nouvelle-autorisation-graph-pour-message-pull-d-ici-janvier"
translationId: article-eb5da8564c33d2e9
translationOf: proofpoint-essentials-november-2026
url: https://rafaelpfister.ch/fr/blog/proofpoint-essentials-novembre-2026-nouvelle-autorisation-graph-pour-message-pull-d-ici-janvier
translationSourceHash: 269840f611e0f96902a1ed742ee1e48ab8d618005a603133463ff394a9a28a3c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:24:07.661Z
translationReview: required
---

# Proofpoint Essentials novembre 2026 : nouvelle autorisation Graph pour Message Pull d’ici janvier 2027

Le 12 novembre 2026, Proofpoint a publié les notes de version de Proofpoint Essentials, le service cloud de sécurité des e-mails destiné aux petites et moyennes entreprises. Cette entrée est la seule version de novembre 2026 et ne contient aucune nouvelle fonctionnalité ni correction de bogue : elle annonce que One-Click Message Pull et Automatic Remediation ne fonctionneront après le 1er janvier 2027 qu’avec l’autorisation Microsoft Graph supplémentaire `Mail-Advanced.ReadWrite.All`. Cette évolution découle d’un changement de Microsoft qui, à partir du 31 décembre 2026, limite la modification de propriétés sensibles des messages envoyés ou reçus via l’API Graph. Cet avis s’adresse aux administrateurs et partenaires qui ont intégré Proofpoint Essentials à Microsoft 365 et utilisent l’une des deux fonctionnalités. Les exploitants de Proofpoint Protection Server (PPS, sur site) ne sont pas concernés par cette version.

## Aperçu des versions

| Version | Date | Type | Points clés |
|---|---|---|---|
| Version du 12 novembre 2026 | 12.11.2026 | Avis relatif aux autorisations | `Mail-Advanced.ReadWrite.All` requis pour One-Click Message Pull et Automatic Remediation à partir du 1er janvier 2027 |

## Autorisation Microsoft Graph Mail-Advanced.ReadWrite.All

Selon les notes de version, les clients qui souhaitent continuer à utiliser One-Click Message Pull et Automatic Remediation après le 1er janvier 2027 doivent étendre leur intégration Microsoft 365 avec l’autorisation d’application `Mail-Advanced.ReadWrite.All` (Microsoft Graph). Les autorisations existantes restent inchangées. Aucune configuration supplémentaire n’est nécessaire dans Proofpoint Essentials.

La cause se situe chez Microsoft. Dans le Microsoft 365 Developer Blog, l’équipe Exchange a annoncé le 26 mars 2026 que, à partir du 31 décembre 2026, les applications ne pourront modifier les propriétés sensibles des messages qui ne sont pas des brouillons qu’avec l’une des autorisations `Mail-Advanced.ReadWrite`, `Mail-Advanced.ReadWrite.All` ou `Mail-Advanced.ReadWrite.Shared`. Microsoft cite notamment l’objet, le corps du message et les destinataires. Avec `Mail.ReadWrite`, il ne sera ensuite possible de modifier ces propriétés que sur les brouillons. Toutes les autorisations Mail-Advanced nécessitent le consentement d’un administrateur du tenant. Selon Microsoft, ces autorisations sont déjà disponibles ; il est donc possible de les ajouter avant la date limite.

La documentation de l’intégration Microsoft 365 de Proofpoint Essentials ne mentionne jusqu’à présent pour One-Click Message Pull que `Mail.ReadWrite` (état au 17 novembre 2026). Avec cette autorisation, Proofpoint Essentials déplace les messages directement dans la boîte aux lettres de l’utilisateur, par exemple vers ou hors de la quarantaine.

## Intégration via l’application Microsoft Integration

Si l’application Azure ou Entra pour Proofpoint Essentials a été créée automatiquement via l’application Microsoft Integration, ajoutez l’autorisation comme suit selon les notes de version :

1. Connectez-vous à Proofpoint Essentials en tant qu’administrateur ayant accès au client concerné (Partner Admin, Channel Admin ou Org Admin).
2. Ouvrez **Account Management > Integrations**.
3. Dans la tuile **Microsoft 365**, cliquez sur **Details**.
4. Cliquez sur **Update Permissions**.
5. Faites confirmer la demande de consentement qui suit par un administrateur Microsoft 365 disposant d’un rôle suffisant, par exemple Global Administrator ou Privileged Role Administrator.

L’application ajoute ensuite l’autorisation `Mail-Advanced.ReadWrite.All`.

## Application Entra créée manuellement

Si l’application a été enregistrée manuellement, les notes de version indiquent la procédure suivante :

1. Connectez-vous au centre d’administration Microsoft Entra (`https://entra.microsoft.com`) avec un compte autorisé à accorder des consentements administrateur (par exemple Global Administrator ou Privileged Role Administrator).
2. Ouvrez **Identity > Applications > App registrations**.
3. Ouvrez l’enregistrement d’application pour Proofpoint Essentials. S’il n’apparaît pas, passez à l’onglet **All applications**.
4. Sélectionnez **API permissions**, puis **Add a permission > Microsoft Graph > Application permissions**.
5. Recherchez `Mail-Advanced.ReadWrite.All`, cochez l’autorisation et sélectionnez **Add permissions**.
6. Sélectionnez **Grant admin consent for <Organisation>** et confirmez.
7. Vérifiez que le statut de l’autorisation affiche **Granted for <Organisation>**.

Selon Proofpoint, toutes les autorisations existantes sont conservées.

## Procédure recommandée

Ajoutez `Mail-Advanced.ReadWrite.All` avant le 31 décembre 2026, date à partir de laquelle Microsoft appliquera la restriction ; Proofpoint indique le 1er janvier 2027 comme date limite. Déterminez d’abord si l’application Entra a été créée via l’application Microsoft Integration ou manuellement, puis suivez la procédure appropriée. Prévoyez pour cela un administrateur disposant du rôle Global Administrator ou Privileged Role Administrator. Après le consentement, vérifiez dans le centre d’administration Entra que l’autorisation est affichée comme accordée. Les notes de version ne précisent pas ce qui se produira exactement après la date limite sans la nouvelle autorisation ; elles indiquent uniquement que celle-ci est requise pour continuer à utiliser les deux fonctionnalités (état au 17 novembre 2026).

## Sources

1.  [Proofpoint Essentials: Release Notes](https://help.proofpoint.com/Essentials/Additional_Resources/Release_Notes): entrée du 12 novembre 2026 avec la nouvelle autorisation, la date limite du 1er janvier 2027 et les étapes pour les deux types d’intégration.

2.  [Proofpoint Essentials: Microsoft 365 Integration MX](https://help.proofpoint.com/Essentials/Product_Documentation/Account_Management/Integrations/Microsoft_365_Integration_MX): autorisations Graph actuelles de l’intégration, dont `Mail.ReadWrite` pour One-Click Message Pull.

3.  [Microsoft 365 Developer Blog: Graph API updates to sensitive email properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/): annonce du 26 mars 2026, application à partir du 31 décembre 2026, autorisations Mail-Advanced avec consentement administrateur.
