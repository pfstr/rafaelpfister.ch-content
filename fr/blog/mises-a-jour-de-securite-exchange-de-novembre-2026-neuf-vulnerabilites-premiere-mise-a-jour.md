---
title: "Mises à jour de sécurité Exchange de novembre 2026 : neuf vulnérabilités, première mise à jour réservée à Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "La SU de novembre corrige neuf vulnérabilités dans Exchange Server SE, dont une exécution de code à distance critique pour laquelle un compte à faibles privilèges suffit. Exchange 2016 et 2019 ne reçoivent plus de mise à jour. La mise à jour Windows de novembre corrige les pannes Exchange survenues après les mises à jour Windows d’octobre."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybride"
timeToRead: "6 min de lecture"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "mises-a-jour-de-securite-exchange-de-novembre-2026-neuf-vulnerabilites-premiere-mise-a-jour"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-de-novembre-2026-neuf-vulnerabilites-premiere-mise-a-jour
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:24:02.807Z
translationReview: automatic
---

# Mises à jour de sécurité Exchange de novembre 2026 : neuf vulnérabilités, première mise à jour réservée à Exchange SE

Microsoft a publié le 10 novembre 2026 une mise à jour de sécurité (SU) pour Exchange Server SE. Selon l’article KB, elle corrige neuf vulnérabilités. Aucune n’était connue publiquement à l’avance et, selon le Security Update Guide, aucune n’est activement exploitée. Microsoft classe une exécution de code à distance comme *Critical* et « Exploitation More Likely » ; un compte à faibles privilèges suffit pour l’exploiter. Ce mois-ci est particulier pour deux raisons : c’est la première SU publiée uniquement pour Exchange SE, et les pannes Exchange après les mises à jour Windows Server d’octobre sont corrigées par les mises à jour Windows de novembre.

## Versions d’Exchange pour lesquelles la mise à jour est disponible

La SU du 10 novembre 2026 n’est disponible que pour une seule version :

- **Exchange Server Subscription Edition (SE) RTM** : KB5133453, build 15.2.2562.57 ; disponible publiquement.

Vous pouvez comparer votre version avec la liste des [numéros de build Exchange](/tools/exchange-builds).

Aucune mise à jour n’est disponible pour Exchange 2016 et 2019. La [SU d’octobre](/blog/exchange-security-updates-oktober-2026) était la dernière, y compris pour les clients du programme ESU Period 2, et ce programme est terminé selon l’annonce. Selon Microsoft, Exchange Online est déjà protégé ; dans les environnements hybrides, chaque serveur Exchange doit néanmoins recevoir la SU, y compris les serveurs de gestion uniquement et les machines équipées des Exchange Management Tools.

## Vue d’ensemble des vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Exécution de code à distance | 8.8 |
| CVE-2026-118149 | Exécution de code à distance | 8.8 |
| CVE-2026-118150 | Élévation de privilèges | 8.8 |
| CVE-2026-118154 | Élévation de privilèges | 8.8 |
| CVE-2026-118153 | Déni de service | 7.5 |
| CVE-2026-118151 | Élévation de privilèges | 7.2 |
| CVE-2026-118157 | Élévation de privilèges | 7.2 |
| CVE-2026-118208 | Usurpation | 7.1 |
| CVE-2026-118152 | Usurpation | 6.5 |

Les neuf failles sont accessibles par le réseau (AV:N) et présentent, selon le vecteur CVSS, une faible complexité d’attaque (AC:L). Le Security Update Guide ne mentionne que Exchange Server SE RTM comme produit concerné.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) est la seule faille du mois que Microsoft classe comme *Critical* (exécution de code à distance, CVSS 8.8), et la seule évaluée comme « Exploitation More Likely ». Selon la FAQ du Security Update Guide, un attaquant authentifié disposant de faibles privilèges peut envoyer une requête spécialement conçue à un serveur concerné et ainsi exécuter du code sur le serveur ; aucune interaction utilisateur n’est nécessaire. Microsoft cite comme cause une validation insuffisante des entrées (CWE-706). Un compte de boîte aux lettres ordinaire suffit donc pour l’attaque, par exemple avec un mot de passe obtenu par phishing.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) est la seconde exécution de code à distance (CVSS 8.8, *Important*). Là aussi, selon la FAQ, un attaquant authentifié envoie une requête spécialement conçue à un service concerné et peut ensuite exécuter du code sur le système cible ; la cause est une injection de code (CWE-94). Microsoft évalue l’exploitation comme « Exploitation Unlikely ».

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (déni de service, CVSS 7.5) est la seule faille qui ne requiert ni compte ni interaction utilisateur. Selon la description, un attaquant non authentifié peut mettre le service hors service via le réseau ; la cause est une complexité algorithmique inefficace (CWE-407). Selon le vecteur, seule la disponibilité est affectée, pas la confidentialité ni l’intégrité.

Les quatre élévations de privilèges diffèrent dans leurs prérequis : pour [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) et [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (toutes deux CVSS 8.8), un compte à faibles privilèges suffit ; pour [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) et [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (toutes deux CVSS 7.2), l’attaquant doit déjà disposer de privilèges élevés. Selon la FAQ, CVE-2026-118151 lui permet d’obtenir les privilèges SYSTEM.

Parmi les deux failles d’usurpation, [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) requiert un compte à faibles privilèges. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) est une falsification de requête intersites : l’attaquant n’a pas besoin de compte, mais un utilisateur doit intervenir (UI:R). Toutes les failles, à l’exception de CVE-2026-118207 et CVE-2026-118149, sont classées comme « Exploitation Less Likely ».

## Exchange 2016 et 2019 : plus de mise à jour, plus de vérification

L’annonce ne mentionne Exchange 2016 et 2019 que dans une note : ces versions ne sont plus testées pour détecter des vulnérabilités et ne recevront plus de mises à jour. Microsoft écrit qu’il faut les considérer comme durablement vulnérables (« permanently vulnerable ») et mettre immédiatement l’organisation à niveau. Le fait que les neuf CVE de novembre ne mentionnent que Exchange SE ne permet donc pas de déterminer si les anciennes versions sont concernées. Frankys Web souligne qu’Exchange SE et Exchange 2019 CU15 sont techniquement proches et qu’au moins une partie des vulnérabilités est probablement également présente dans Exchange 2019.

Pour l’exploitation, cela signifie :

1. **Exchange 2019 CU15 et antérieur, Exchange 2016** : la version reste figée à la SU d’octobre (avec ESU) ou à une version antérieure. Les neuf vulnérabilités de novembre restent ouvertes sur ces serveurs si elles y sont présentes, et Microsoft ne le vérifie plus.

2. **Migration** : Microsoft recommande de passer immédiatement à Exchange SE.

3. **Environnements hybrides** : Exchange Online limite et bloque le flux de messagerie provenant de serveurs On-Premises obsolètes ; les règles sont indiquées dans l’[article sur le transport enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problèmes corrigés

**Pannes Exchange après les mises à jour Windows Server d’octobre** (toutes versions). Le problème avec `MapiExceptionMdbOffline` (ec=1142) n’est pas corrigé par la SU Exchange : selon l’annonce, les mises à jour Windows Server de novembre 2026 doivent être installées. Les notes de publication de ces mises à jour répertorient le correctif sous « [Exchange Server] » et indiquent le problème comme résolu avec les mises à jour à partir du 10 novembre 2026 : KB5131080 pour Windows Server 2025, KB5131090 pour Windows Server 2022, KB5131085 pour Windows Server 2019 et KB5131291 pour Windows Server 2016. Selon l’annonce, l’ordre dans lequel vous installez la SU Exchange et la mise à jour Windows n’a pas d’importance. Si vous avez défini la valeur de registre `EpmapSavedContextCap` depuis l’[article sur les pannes Exchange](/blog/exchange-windows-updates-oktober-2026), vous pouvez la conserver ; selon Microsoft, elle n’est plus nécessaire après l’installation des mises à jour de novembre pour Windows Server.

**Affichage incorrect des disponibilités libre/occupé via Microsoft Graph dans les environnements hybrides** (Exchange SE uniquement). Selon [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867), les utilisateurs disposant d’une boîte aux lettres On-Premises ne voyaient pas d’informations libre/occupé correctes pour les boîtes aux lettres Exchange Online dans Outlook classique lorsque l’API Graph était activée ; seuls des créneaux horaires étaient affichés. L’affichage était correct avec Exchange Web Services (EWS) au lieu de l’API Graph. Le problème est corrigé avec KB5133453 ou une mise à jour plus récente. L’article de support ne mentionne aucune action corrective. Si vous êtes revenu à EWS en raison de cette erreur, vérifiez après l’installation la requête libre/occupé d’une boîte aux lettres Exchange Online avant de réactiver l’API Graph.

## Problèmes connus

Ni l’annonce ni KB5133453 ne mentionnent de problèmes connus pour la SU de novembre (état au 11 novembre 2026). L’annonce ne comporte aucun commentaire de l’équipe Exchange, et Microsoft indique que la documentation peut encore être incomplète au moment de la publication. Aucun problème connu des mois précédents ne reste ouvert : les problèmes des mises à jour de septembre sont résolus depuis la SU d’octobre, et le problème Windows d’octobre est corrigé par les mises à jour de novembre pour Windows Server (voir ci-dessus).

## Installation et suivi

Microsoft décrit dans l’annonce la procédure habituelle : inventorier avec l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) les serveurs qui nécessitent des mises à jour ou des étapes manuelles, installer la SU, redémarrer le serveur et vérifier que tous les services Exchange fonctionnent. Si des services sont désactivés après l’installation, celle-ci a été interrompue. En cas d’erreurs pendant ou après l’installation, Microsoft renvoie au [script SetupAssist](https://aka.ms/ExSetupAssist). Après l’installation, le Health Checker indique selon KB5133453 si la SU est correctement installée et si d’autres étapes sont nécessaires.

Selon KB5133453, le fichier d’installation s’appelle `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center : « SU12 »). La SU remplace la SU d’octobre KB5128895. Les SU sont cumulatives : si une SU plus ancienne est installée sur Exchange SE RTM, installez directement la SU de novembre.

Pour les environnements hybrides, la FAQ précise que si le certificat d’authentification est modifié après l’installation d’une SU, il faut exécuter à nouveau le Hybrid Configuration Wizard. Sous Windows Server 2025, les SU Exchange installées n’apparaissent pas dans le Panneau de configuration ; Microsoft déconseille de désinstaller les SU.

Les actions de suivi suivantes des mois précédents restent à effectuer si ce n’est pas déjà fait : supprimer l’override libre/occupé `EnableRouteThroughMSGraphFeature` (voir l’[article d’octobre](/blog/exchange-security-updates-oktober-2026)), supprimer le SettingOverride Wrapper `DisableBlockSharedAndUserMailboxHeaders` (voir l’[article de septembre](/blog/exchange-security-updates-september-2026)) et vérifier si la mitigation CVE-2026-42897 (M2.1.0) est toujours active (voir l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026)).

## Procédure recommandée

Installez rapidement la SU de novembre sur tous les serveurs équipés d’Exchange SE ainsi que sur les machines avec les Management Tools : pour l’exécution de code à distance critique CVE-2026-118207, un compte à faibles privilèges suffit, et Microsoft estime son exploitation plus probable. Installez les mises à jour de novembre pour Windows Server pendant la même fenêtre de maintenance afin de corriger la cause des pannes d’octobre. Il n’existe plus de correctif pour Exchange 2016 et 2019 ; selon Microsoft, ces serveurs doivent être considérés comme durablement vulnérables et remplacés immédiatement par Exchange SE.

## Sources

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368) : annonce du 10 novembre 2026 avec la version prise en charge, une note sur Exchange 2016/2019, le problème Windows corrigé et la FAQ (aucun commentaire de l’équipe Exchange, état au 11 novembre 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453) : liste des CVE, problème corrigé, fichier d’installation, mise à jour remplacée et note sur le Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/) : type, CVSS, gravité, évaluation de l’exploitabilité, FAQ et produit concerné pour les neuf CVE de novembre.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates) : numéro de build de la SU de novembre.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867) : symptômes et correctif avec KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382) : publication de l’équipe Exchange sur le problème Windows d’octobre auquel l’annonce renvoie.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080) : notes de publication pour Windows Server 2025 avec le correctif Exchange et le problème connu marqué comme résolu.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090) : notes de publication pour Windows Server 2022 avec le même correctif.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085) : notes de publication pour Windows Server 2019 avec le même correctif.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291) : notes de publication pour Windows Server 2016 avec le même correctif.

11. [Nouvelles mises à jour de sécurité pour Exchange Server (novembre 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/) : analyse en allemand des neuf CVE et évaluation pour Exchange 2016 et 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker) : inventaire des versions CU/SU avant et après l’installation.
