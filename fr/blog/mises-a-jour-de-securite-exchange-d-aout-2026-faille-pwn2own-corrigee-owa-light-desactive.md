---
title: "Mises à jour de sécurité Exchange d’août 2026 : vulnérabilité Pwn2Own corrigée, OWA Light désactivé"
navTitle: "Exchange SU 08/2026"
description: "La SU d’août corrige sept vulnérabilités, dont l’exploit Exchange démontré lors de Pwn2Own 2026, et désactive définitivement OWA Light. Microsoft explique également pourquoi les SU Exchange paraissent désormais chaque mois et pourquoi Exchange SE CU1 se fait toujours attendre."
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
translationSourceHash: fcce9c9e87a2b2cd669e4d2647b79283a9e29727194593a7208e977c56763c97
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:51:25.342Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-d-aout-2026-faille-pwn2own-corrigee-owa-light-desactive
---

# Mises à jour de sécurité Exchange d’août 2026 : vulnérabilité Pwn2Own corrigée, OWA Light désactivé

Le 11 août 2026, Microsoft a publié des mises à jour de sécurité (SU) pour Exchange, pour le quatrième mois consécutif. Ces mises à jour corrigent sept vulnérabilités. Aucune n’était connue publiquement à l’avance, aucune n’est activement exploitée à l’heure actuelle, et Microsoft classe l’exploitation des sept vulnérabilités comme « Exploitation Less Likely ». Il ne s’agit néanmoins pas d’un Patch Tuesday de routine, pour trois raisons : la mise à jour corrige la vulnérabilité Exchange démontrée lors du concours de piratage Pwn2Own, elle **désactive définitivement OWA Light après presque vingt ans**, et l’équipe Exchange a ensuite expliqué pourquoi le rythme mensuel reste pour l’instant la norme.

## Pour quelles versions d’Exchange la mise à jour est disponible

Les SU sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5121573, build 15.2.2562.46 ; en tant que mise à jour publique normalement disponible.
- **Exchange Server 2019 CU15** : KB5121574, build 15.2.1748.49 ; uniquement via le **programme ESU Period 2**.
- **Exchange Server 2019 CU14** : KB5121575, build 15.2.1544.44 ; uniquement via ESU Period 2.
- **Exchange Server 2016 CU23** : KB5121576, build 15.1.2507.72 ; uniquement via ESU Period 2.

La situation est la même qu’en juillet : Exchange 2016 et 2019 ne sont plus pris en charge. Seules les personnes inscrites au programme ESU Period 2 recevront les SU de mai à octobre 2026. Tous les autres restent non corrigés, avec désormais quatorze vulnérabilités ouvertes, dont certaines sont sévèrement notées ; la migration vers Exchange SE ne tolère plus aucun délai. Exchange Online est déjà protégé ; dans les environnements hybrides, la SU doit néanmoins être installée sur tous les serveurs Exchange, y compris les serveurs de gestion dédiés et les machines où seuls les Exchange Management Tools sont installés.

Le problème connu des *messages wrapper* dans les boîtes aux lettres partagées des environnements hybrides persiste avec la SU d’août ; selon Microsoft, le correctif est prévu dans une prochaine mise à jour. Une précision rassurante provient toutefois des commentaires de l’annonce de publication : les personnes ayant appliqué le SettingOverride documenté comme solution de contournement ne doivent **pas** le recréer après l’installation de la SU d’août. La mise à jour laisse l’override intact, comme l’équipe Exchange l’a confirmé.

## Aperçu des sept vulnérabilités

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

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** affiche le score mensuel le plus élevé, avec un CVSS de 8.8 : une exécution de code à distance qu’un attaquant authentifié disposant de faibles privilèges peut déclencher sans aucune interaction de l’utilisateur. N’importe quel compte de boîte aux lettres compromis suffit comme point de départ ; à l’ère du phishing et du credential stuffing, « authentifié » n’est pas un obstacle majeur.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** est la seule vulnérabilité du mois que Microsoft classe comme *Critical* (élévation de privilèges, CVSS 8.0). Cette référence cache davantage d’histoire que le numéro ne le laisse penser : interrogée sur la correction de l’exploit Exchange démontré par Orange Tsai lors de **Pwn2Own 2026**, l’équipe Exchange renvoie précisément à cette CVE dans les commentaires de l’annonce de publication. La découverte du concours est donc corrigée : une raison supplémentaire de ne pas laisser la SU d’août de côté, car les techniques Pwn2Own sont généralement publiées en détail à l’expiration des délais de confidentialité. C’est désormais le cas : un proof of concept est public, et le BSI signale environ 85 % de serveurs On-Premises vulnérables en Allemagne. Le [article détaillé sur CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay) explique le déroulement technique de l’attaque (MRSProxy sans Channel Binding, relais NTLM) ainsi que le contexte de ces chiffres.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (usurpation, CVSS 7.3) est la raison directe de la désactivation d’OWA Light, comme expliqué ci-dessous.

Les autres vulnérabilités : CVE-2026-62910 (EoP, 7.2) requiert déjà des privilèges élevés ; CVE-2026-62912 (DoS), CVE-2026-62915 (contournement de fonctionnalité de sécurité) et CVE-2026-65813 (EoP) affichent un CVSS de 6.5. Comme d’habitude, les détails figurent dans le Security Update Guide (filtre « Server Software » pour Exchange SE ou « ESU » pour 2016/2019).

## OWA Light : la fin après presque vingt ans

### Ce que la mise à jour change

Avec l’installation de la SU d’août, **OWA Light est désactivé de manière permanente** : sur chaque serveur recevant cette mise à jour, ou une version ultérieure. Les personnes qui ouvrent l’interface Light seront désormais redirigées vers Outlook on the web normal. La désactivation fait partie intégrante de la mise à jour et ne peut pas être annulée par un paramètre ; Microsoft l’avait annoncée quelques semaines auparavant dans un billet de blog distinct.

OWA Light remonte à l’époque d’Exchange 2007 : une interface web volontairement simplifiée, utilisée comme solution de repli pour les anciens navigateurs et les connexions lentes, officiellement dépréciée depuis août 2024. La justification de sa suppression est liée à la sécurité : un chemin de rendu Legacy distinct, parallèlement à OWA moderne, augmente la complexité et donc la surface d’attaque ; CVE-2026-62914 en apporte la preuve concrète. Les personnes ayant lu l’[article de juillet](/blog/exchange-security-updates-juli-2026) s’en souviennent également : la mitigation CVE-2026-42897 de mai avait déjà rendu OWA Light inutilisable au passage. L’interface était donc déjà sur la sellette.

### Pour ceux qui ne peuvent pas appliquer le correctif : désactiver OWA Light manuellement

Point important pour toutes les personnes qui ne peuvent pas (encore) installer la SU d’août, par exemple parce que l’activation ESU fait défaut : Microsoft recommande expressément de **désactiver manuellement** OWA Light dans ce cas afin d’atténuer CVE-2026-62914. Cela se fait via la stratégie de boîte aux lettres OWA et la page de connexion :

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

La première commande désactive la version Light pour toutes les boîtes aux lettres de la stratégie concernée ; la seconde supprime l’option « Utiliser la version Light » de la page de connexion OWA. Les modifications du répertoire virtuel OWA ne prennent effet de manière fiable qu’après le recyclage du pool d’applications OWA ou un `iisreset`.

### Ce que les administrateurs devraient vérifier maintenant

La désactivation est techniquement triviale, mais pas toujours sur le plan organisationnel : OWA Light était la solution de repli discrète pour des scénarios de niche. Il convient désormais de vérifier les favoris et les instructions du helpdesk qui intègrent `?layout=light` en dur, les appareils de type kiosque et terminaux équipés d’anciens navigateurs, ainsi que les instructions internes destinées aux utilisatrices et utilisateurs qui employaient la version Light pour des raisons d’accessibilité. Outlook on the web moderne fonctionne dans tous les navigateurs actuels et dispose de ses propres fonctions d’accessibilité ; mais sans information préalable des utilisateurs concernés, les tickets afflueront.

## Pourquoi une SU paraît désormais chaque mois et où en est Exchange SE CU1

Deux jours après la publication, l’équipe Exchange a répondu dans un billet de blog remarquablement transparent (« Where is Exchange SE CU1 anyway? ») à la question que se posent de nombreux administrateurs. En bref : Microsoft utilise des outils d’IA à l’échelle du groupe pour trouver des vulnérabilités dans ses propres produits. Les équipes, Exchange compris, traitent actuellement les découvertes signalées : validation, reproduction, correction, tests de régression et publication mensuelle. Depuis mai 2026, une SU Exchange paraît ainsi chaque mois, et Microsoft indique explicitement que ce rythme accru va perdurer.

Le très attendu **CU1 pour Exchange SE** est retardé précisément pour cette raison. Initialement annoncé pour le premier semestre 2026, puis reporté au second, il n’a désormais plus aucune date cible. Microsoft ne souhaite publier CU1 que lorsqu’un mois sans livraison de sécurité urgente s’intercale ; un CU immédiatement dépassé par une SU imposerait un double travail de mise à jour à de nombreuses organisations. D’ici là, la charge de sécurité mensuelle est intégrée en continu dans le build interne de CU1.

Dans la pratique, cela signifie deux choses. Premièrement : attendre CU1 n’est pas une stratégie, ni pour la migration vers SE ni pour l’installation des SU. Deuxièmement : une **fenêtre de maintenance mensuelle** pour Exchange doit désormais faire partie intégrante du calendrier d’exploitation, comme c’est depuis longtemps une évidence pour les serveurs Windows.

## Installation et suivi

La procédure reste éprouvée : commencez par inventorier, avec le [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), quels serveurs disposent de quelle version CU/SU et quelles étapes manuelles restent en attente. Installez ensuite la SU (en cas de CU obsolète, l’[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) indique le chemin), redémarrez le serveur et vérifiez que tous les services Exchange ont démarré correctement. Si des services sont *désactivés*, l’installation a été interrompue ; la solution de contournement documentée dans l’article de support Microsoft concernant l’erreur de version de fichier, ou le [script SetupAssist](https://aka.ms/ExSetupAssist), peut alors aider. Enfin, exécutez à nouveau le Health Checker.

Les SU sont cumulatives : si vous avez ignoré la SU de juillet, vous pouvez installer directement celle d’août. Et pour les environnements hybrides, la règle habituelle s’applique : si le certificat Auth est modifié après l’installation de la SU, le Hybrid Configuration Wizard doit être exécuté à nouveau.

Un travail de suivi de juillet reste d’actualité : les personnes ayant encore activé la mitigation CVE-2026-42897 (M2.1.0) devraient maintenant la supprimer ; l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026) explique comment procéder correctement.

## Procédure recommandée

En résumé : installez rapidement la SU d’août sur tous les serveurs Exchange et les machines équipées des Management Tools : la vulnérabilité Pwn2Own et la RCE à 8.8 suffisent à justifier de ne pas attendre le prochain Patch Tuesday. Si l’application du correctif n’est pas immédiatement possible : OWA Light peut être désactivé manuellement comme mesure immédiate contre CVE-2026-62914. Avant la désactivation d’OWA Light, identifiez et informez les groupes d’utilisateurs concernés (anciens favoris, navigateurs de kiosque, flux de travail d’accessibilité). Exécutez ensuite le Health Checker, réalisez les travaux de suivi en attente depuis juillet et prévoyez une fenêtre de maintenance Exchange mensuelle, car le rythme perdure.

## Sources

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951) : annonce officielle de publication avec les versions prises en charge, la note concernant OWA Light, les problèmes connus et la FAQ ; les commentaires confirment le correctif Pwn2Own (CVE-2026-62911) ainsi que le maintien du Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943) : annonce préalable de la désactivation, accompagnée de la recommandation de Microsoft de désactiver manuellement OWA Light en l’absence de correctif.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837) : l’équipe Exchange sur la recherche de vulnérabilités assistée par l’IA, le rythme mensuel durable des SU et le retard de CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates) : référence pour les numéros de build des SU d’août.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603) : conditions et durée (de mai à octobre 2026) du programme ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719) : problème hybride connu depuis juin, avec la solution de contournement SettingOverride.

7.  [Nouvelles mises à jour de sécurité pour Exchange Server (août 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/) : présentation en allemand des sept CVE, avec leurs scores CVSS et builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy) : le paramètre `OWALightEnabled` permettant de désactiver manuellement la version Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker) : inventaire des versions CU/SU et des étapes manuelles restantes avant et après l’installation.
