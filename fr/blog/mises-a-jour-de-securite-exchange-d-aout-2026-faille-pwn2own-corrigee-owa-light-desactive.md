---
title: "Mises à jour de sécurité Exchange d’août 2026 : faille Pwn2Own corrigée, OWA Light désactivé"
navTitle: "Exchange SU 08/2026"
description: "La mise à jour de sécurité d’août corrige sept vulnérabilités, dont l’exploit Exchange démontré lors de Pwn2Own 2026, et désactive définitivement OWA Light. Microsoft explique également pourquoi les mises à jour de sécurité Exchange paraissent désormais chaque mois et pourquoi Exchange SE CU1 se fait toujours attendre."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min de lecture"
themen:
  - exchange-updates
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "mises-a-jour-de-securite-exchange-d-aout-2026-faille-pwn2own-corrigee-owa-light-desactive"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
translationSourceHash: 41e10101798a88902017688d719457fce48959ba3acd2b3f1c757867b1b368d7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:58:25.983Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-d-aout-2026-faille-pwn2own-corrigee-owa-light-desactive
---

# Mises à jour de sécurité Exchange d’août 2026 : faille Pwn2Own corrigée, OWA Light désactivé

Le 11 août 2026, Microsoft a publié des mises à jour de sécurité (SU) pour Exchange Server, pour le quatrième mois consécutif. Les mises à jour corrigent sept vulnérabilités. Aucune n’était connue publiquement à l’avance, aucune n’est activement exploitée selon les informations actuelles, et Microsoft classe l’exploitation des sept comme « Exploitation Less Likely ». Il ne s’agit toutefois pas d’un Patch Tuesday de routine, pour trois raisons : la mise à jour corrige la faille Exchange démontrée lors du concours de piratage Pwn2Own, elle **désactive définitivement OWA Light après près de vingt ans**, et l’équipe Exchange a ensuite expliqué pourquoi le rythme mensuel reste pour l’instant la norme.

## Pour quelles versions d’Exchange la mise à jour est disponible

Les SU sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5121573, build 15.2.2562.46 ; en tant que mise à jour publique disponible normalement.
- **Exchange Server 2019 CU15** : KB5121574, build 15.2.1748.49 ; uniquement via le **programme ESU de période 2**.
- **Exchange Server 2019 CU14** : KB5121575, build 15.2.1544.44 ; uniquement via ESU période 2.
- **Exchange Server 2016 CU23** : KB5121576, build 15.1.2507.72 ; uniquement via ESU période 2.

La situation est la même qu’en juillet : Exchange 2016 et 2019 ne sont plus pris en charge. Seules les personnes inscrites au programme ESU de période 2 reçoivent les SU de mai à octobre 2026. Tous les autres restent sans correctif, avec désormais quatorze vulnérabilités ouvertes, dont certaines à gravité élevée ; la migration vers Exchange SE ne souffre plus aucun report dans ce cas. Exchange Online est déjà protégé ; dans les environnements hybrides, le SU doit néanmoins être installé sur tous les serveurs Exchange, y compris les serveurs dédiés à l’administration et les machines sur lesquelles seuls les Exchange Management Tools sont installés.

Le problème connu des *messages wrapper* dans les boîtes aux lettres partagées d’environnements hybrides persiste également avec le SU d’août ; selon Microsoft, le correctif est prévu dans une prochaine mise à jour. Une clarification rassurante est toutefois venue des commentaires de l’annonce de publication : les personnes ayant configuré le SettingOverride documenté comme solution de contournement ne doivent **pas** le recréer après l’installation du SU d’août. Comme l’a confirmé l’équipe Exchange, la mise à jour ne modifie pas l’override.

## Vue d’ensemble des sept vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Exécution de code à distance | 8.8 |
| CVE-2026-62911 | Élévation de privilèges | 8.0 |
| CVE-2026-62914 | Usurpation | 7.3 |
| CVE-2026-62910 | Élévation de privilèges | 7.2 |
| CVE-2026-62912 | Déni de service | 6.5 |
| CVE-2026-62915 | Contournement de fonctionnalité de sécurité | 6.5 |
| CVE-2026-65813 | Élévation de privilèges | 6.5 |

Trois d’entre elles méritent un examen plus attentif.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** affiche le score le plus élevé du mois, avec un CVSS de 8.8 : une exécution de code à distance qu’un attaquant authentifié disposant de droits limités peut déclencher sans aucune interaction de l’utilisateur. N’importe quel compte de boîte aux lettres compromis suffit comme point de départ ; à l’ère du phishing et du credential stuffing, « authentifié » ne constitue pas un obstacle élevé.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** est la seule vulnérabilité du mois que Microsoft classe comme *Critical* (élévation de privilèges, CVSS 8.0). Son histoire est plus riche que ne le laisse entendre ce numéro sobre : à la question de savoir si l’exploit Exchange démontré par Orange Tsai lors de **Pwn2Own 2026** avait entre-temps été corrigé, l’équipe Exchange renvoie précisément à cette CVE dans les commentaires de l’annonce de publication. La découverte du concours est donc corrigée : une raison supplémentaire de ne pas laisser le SU d’août de côté, car les techniques Pwn2Own sont habituellement publiées en détail après l’expiration des délais d’embargo.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (usurpation, CVSS 7.3) est la cause directe de la désactivation d’OWA Light, comme nous le verrons juste après.

Les autres failles : CVE-2026-62910 (EoP, 7.2) exige déjà des droits élevés, tandis que CVE-2026-62912 (DoS), CVE-2026-62915 (contournement de fonctionnalité de sécurité) et CVE-2026-65813 (EoP) obtiennent un CVSS de 6.5. Les détails figurent comme d’habitude dans le Security Update Guide (filtre « Server Software » pour Exchange SE ou « ESU » pour 2016/2019).

## OWA Light : fin de parcours après près de vingt ans

### Ce que change la mise à jour

L’installation du SU d’août **désactive définitivement OWA Light** : sur chaque serveur recevant cette mise à jour ou une version ultérieure. Les personnes qui ouvrent l’interface Light seront désormais redirigées vers Outlook on the web normal. La désactivation fait partie intégrante de la mise à jour et ne peut pas être annulée par un paramètre ; Microsoft l’avait annoncée quelques semaines auparavant dans un billet de blog distinct.

OWA Light remonte à l’époque d’Exchange 2007 : une interface web volontairement simplifiée, conçue comme solution de repli pour les anciens navigateurs et les connexions lentes, officiellement déconseillée depuis août 2024. La raison de sa suppression est liée à la sécurité : un chemin de rendu hérité distinct, parallèlement à OWA moderne, accroît la complexité et donc la surface d’attaque ; CVE-2026-62914 en apporte la preuve concrète. Les personnes ayant lu [l’article de juillet](/blog/exchange-security-updates-juli-2026) s’en souviennent également : la mitigation de mai pour CVE-2026-42897 avait déjà rendu OWA Light inutilisable au passage. L’interface était donc déjà condamnée.

### Pour celles et ceux qui ne peuvent pas appliquer le correctif : désactiver OWA Light manuellement

Important pour toutes les personnes qui ne peuvent pas (encore) installer le SU d’août, par exemple faute d’activation ESU : Microsoft recommande expressément dans ce cas de **désactiver manuellement OWA Light** afin d’atténuer CVE-2026-62914. Cela se fait via la stratégie de boîte aux lettres OWA et la page de connexion :

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

La première commande désactive la version Light pour toutes les boîtes aux lettres de la stratégie concernée, la seconde supprime le choix « Utiliser la version Light » de la page de connexion OWA. Les modifications apportées au répertoire virtuel OWA ne prennent effet de manière fiable qu’après un recyclage du pool d’applications OWA ou un `iisreset`.

### Ce que les administrateurs devraient maintenant vérifier

La désactivation est techniquement triviale, mais pas toujours sur le plan organisationnel : OWA Light constituait la solution de repli discrète pour des scénarios de niche. Il convient désormais de vérifier les favoris et instructions du helpdesk qui ont `?layout=light` codé en dur, les appareils kiosque et terminaux équipés d’anciens navigateurs, ainsi que les guides internes destinés aux utilisatrices et utilisateurs ayant employé la version Light pour des raisons d’accessibilité. Outlook on the web moderne fonctionne dans tous les navigateurs actuels et propose ses propres fonctionnalités d’accessibilité ; mais sans information préalable des utilisateurs concernés, les tickets seront inévitables.

## Pourquoi un SU paraît désormais chaque mois et où en est Exchange SE CU1

Deux jours après la publication, l’équipe Exchange a répondu dans un billet de blog remarquablement transparent (« Where is Exchange SE CU1 anyway? ») à la question que se posent de nombreux administrateurs. En bref : Microsoft utilise à l’échelle du groupe des outils d’IA pour détecter des vulnérabilités dans ses propres produits. Les équipes, dont celle d’Exchange, traitent actuellement les découvertes signalées : validation, reproduction, correction, tests de régression et livraison mensuelle. Depuis mai 2026, un SU Exchange paraît ainsi chaque mois, et Microsoft l’affirme explicitement : ce rythme soutenu va se poursuivre.

Le très attendu **CU1 pour Exchange SE** est retardé précisément pour cette raison. Initialement annoncé pour le premier semestre 2026, puis reporté au second, il n’existe désormais plus de date cible. Microsoft ne veut publier CU1 que lorsqu’un mois sans livraison urgente de sécurité s’intercale ; un CU immédiatement dépassé par un SU occasionnerait un double travail de mise à jour pour de nombreuses organisations. Entre-temps, la charge de sécurité mensuelle est continuellement intégrée à la build interne de CU1.

Dans la pratique, cela signifie deux choses. Premièrement : attendre CU1 n’est pas une stratégie, ni pour la migration vers SE ni pour l’installation des SU. Deuxièmement : une **fenêtre de maintenance mensuelle** pour Exchange doit désormais faire partie intégrante du calendrier d’exploitation, comme c’est depuis longtemps une évidence pour les serveurs Windows.

## Installation et suivi

La procédure reste éprouvée : commencer par inventorier avec le [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) quels serveurs se trouvent à quel niveau CU/SU et si des étapes manuelles sont encore en attente. Installer ensuite le SU (en cas de niveau CU obsolète, le [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) indique le chemin), redémarrer le serveur et vérifier que tous les services Exchange ont démarré correctement. Si des services sont *désactivés*, l’installation a été interrompue ; dans ce cas, la solution de contournement documentée dans l’article de support Microsoft relatif à l’erreur de version de fichier ou le [script SetupAssist](https://aka.ms/ExSetupAssist) peut aider. Pour terminer, exécuter à nouveau le Health Checker.

Les SU sont cumulatives : les personnes ayant ignoré le SU de juillet peuvent installer directement celui d’août. Et pour les environnements hybrides, la règle complémentaire connue s’applique : si le certificat d’authentification est remplacé après l’installation du SU, il convient d’exécuter à nouveau le Hybrid Configuration Wizard.

Un travail de suivi de juillet reste d’actualité : les personnes qui ont encore activée la mitigation CVE-2026-42897 (M2.1.0) devraient désormais la supprimer ; la procédure correcte est décrite dans [l’article sur le SU de juillet](/blog/exchange-security-updates-juli-2026).

## Procédure recommandée

En résumé : installer rapidement le SU d’août sur tous les serveurs Exchange et les machines équipées des Management Tools : la faille Pwn2Own et la RCE à 8.8 sont des raisons suffisantes pour ne pas attendre le prochain Patch Tuesday. Les personnes ne pouvant pas appliquer immédiatement le correctif peuvent désactiver manuellement OWA Light comme mesure immédiate contre CVE-2026-62914. Avant la désactivation d’OWA Light, identifier et informer les groupes d’utilisateurs concernés (anciens favoris, navigateurs de kiosques, workflows d’accessibilité). Exécuter ensuite le Health Checker, effectuer les travaux de suivi encore ouverts depuis juillet et planifier une fenêtre de maintenance Exchange mensuelle, car le rythme perdure.

## Sources

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951) : Annonce officielle de publication avec les versions prises en charge, la note concernant OWA Light, les problèmes connus et la FAQ ; les commentaires confirment le correctif Pwn2Own (CVE-2026-62911) et le maintien du SettingOverride pour les messages wrapper.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943) : L’annonce préalable de la désactivation, ainsi que la recommandation de Microsoft de désactiver manuellement OWA Light en l’absence de correctif.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837) : L’équipe Exchange sur la recherche de vulnérabilités assistée par IA, le rythme mensuel soutenu des SU et le retard de CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates) : Référence pour les numéros de build des SU d’août.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603) : Conditions et durée (de mai à octobre 2026) du programme ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719) : Le problème hybride connu depuis juin et sa solution de contournement SettingOverride.

7.  [Nouvelles mises à jour de sécurité pour Exchange Server (août 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/) : Présentation en allemand des sept CVE avec les scores CVSS et les builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy) : Le paramètre `OWALightEnabled` pour la désactivation manuelle de la version Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker) : Inventaire des niveaux CU/SU et des étapes manuelles en attente avant et après l’installation.
