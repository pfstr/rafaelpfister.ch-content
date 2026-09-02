---
title: "Mises à jour de sécurité Exchange d’août 2026 : faille Pwn2Own corrigée, OWA Light désactivé"
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
translationSourceHash: 4c2345cf2955df229b8713cf288ec21bba3e1bd43aef297ecad12536e9bf459a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:54:36.926Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/mises-a-jour-de-securite-exchange-d-aout-2026-faille-pwn2own-corrigee-owa-light-desactive
---

# Mises à jour de sécurité Exchange d’août 2026 : faille Pwn2Own corrigée, OWA Light désactivé

Microsoft a publié le 11 août 2026 des mises à jour de sécurité (SU) pour Exchange Server, pour le quatrième mois consécutif. Les mises à jour corrigent sept vulnérabilités. Aucune n’était connue publiquement à l’avance, aucune n’est activement exploitée à ce jour, et Microsoft évalue pour les sept que l’exploitation est « Exploitation Less Likely ». Il ne s’agit pourtant pas d’un Patch Tuesday ordinaire, pour trois raisons : la mise à jour corrige la faille Exchange démontrée lors du concours de hacking Pwn2Own, elle **désactive définitivement OWA Light après près de vingt ans**, et l’équipe Exchange a ensuite expliqué pourquoi le rythme mensuel restera pour l’instant la norme.

## Pour quelles versions d’Exchange la mise à jour est disponible

Les SU sont disponibles pour les versions suivantes :

- **Exchange Server Subscription Edition (SE) RTM** : KB5121573, build 15.2.2562.46 ; mise à jour publique disponible de manière standard.
- **Exchange Server 2019 CU15** : KB5121574, build 15.2.1748.49 ; uniquement via le **programme ESU période 2**.
- **Exchange Server 2019 CU14** : KB5121575, build 15.2.1544.44 ; uniquement via ESU période 2.
- **Exchange Server 2016 CU23** : KB5121576, build 15.1.2507.72 ; uniquement via ESU période 2.

La situation est identique à celle de juillet : Exchange 2016 et 2019 ne sont plus pris en charge. Seules les personnes inscrites au programme ESU période 2 recevront les SU de mai à octobre 2026. Tous les autres restent non corrigés, avec désormais quatorze vulnérabilités ouvertes, dont certaines à gravité élevée ; la migration vers Exchange SE ne peut plus être différée. Exchange Online est déjà protégé ; dans les environnements hybrides, la SU doit néanmoins être installée sur tous les serveurs Exchange, y compris les serveurs de gestion purs et les machines sur lesquelles seuls les Exchange Management Tools sont installés.

Le problème connu des *messages wrapper* dans les boîtes aux lettres partagées des environnements hybrides persiste avec la SU d’août ; selon Microsoft, le correctif est prévu dans une prochaine mise à jour. Au moins, les commentaires de l’annonce de publication apportent une clarification : les personnes ayant appliqué le SettingOverride documenté comme solution de contournement ne doivent **pas** le recréer après l’installation de la SU d’août. La mise à jour laisse l’override intact, comme l’a confirmé l’équipe Exchange.

## Aperçu des sept vulnérabilités

| CVE | Type | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Exécution de code à distance | 8.8 |
| CVE-2026-62911 | Élévation de privilèges | 8.0 |
| CVE-2026-62914 | Usurpation d’identité | 7.3 |
| CVE-2026-62910 | Élévation de privilèges | 7.2 |
| CVE-2026-62912 | Déni de service | 6.5 |
| CVE-2026-62915 | Contournement de fonctionnalité de sécurité | 6.5 |
| CVE-2026-65813 | Élévation de privilèges | 6.5 |

Trois d’entre elles méritent un examen plus approfondi.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** affiche avec un CVSS de 8.8 la valeur la plus élevée du mois : une exécution de code à distance qu’un attaquant authentifié disposant de droits limités peut déclencher sans aucune interaction utilisateur. N’importe quel compte de boîte aux lettres compromis suffit comme point de départ ; à l’ère du phishing et du credential stuffing, « authentifié » n’est pas un obstacle majeur.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** est la seule vulnérabilité du mois que Microsoft classe comme *Critical* (élévation de privilèges, CVSS 8.0). Son histoire est plus riche que ne le laisse supposer ce numéro sobre : interrogée sur la correction de l’exploit Exchange démontré par Orange Tsai à **Pwn2Own 2026**, l’équipe Exchange renvoie précisément à cette CVE dans les commentaires de l’annonce de publication. La découverte du concours est donc corrigée : une raison supplémentaire de ne pas laisser la SU d’août de côté, car les techniques Pwn2Own sont habituellement publiées en détail après l’expiration des périodes d’embargo. C’est désormais le cas : un proof of concept est public, et le BSI signale environ 85 % de serveurs On-Premises vulnérables en Allemagne. Le déroulement technique de l’attaque (MRSProxy sans Channel Binding, relais NTLM) et l’explication des chiffres sont détaillés dans l’[article complet sur CVE-2026-62911](/blog/cve-2026-62911-exchange-ntlm-relay).

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (usurpation d’identité, CVSS 7.3) est la raison directe de la désactivation d’OWA Light, comme nous le verrons ci-dessous.

Les autres failles : CVE-2026-62910 (EoP, 7.2) exige déjà des droits élevés, tandis que CVE-2026-62912 (DoS), CVE-2026-62915 (contournement de fonctionnalité de sécurité) et CVE-2026-65813 (EoP) obtiennent un CVSS de 6.5. Les détails figurent comme d’habitude dans le Security Update Guide (filtre « Server Software » pour Exchange SE ou « ESU » pour 2016/2019).

## OWA Light : fin de parcours après près de vingt ans

### Ce que change la mise à jour

Avec l’installation de la SU d’août, **OWA Light est désactivé définitivement** : sur chaque serveur recevant cette mise à jour, ou une version ultérieure. Les utilisateurs qui ouvrent l’interface Light seront désormais redirigés vers Outlook on the web standard. Cette désactivation fait partie intégrante de la mise à jour et ne peut pas être annulée au moyen d’un commutateur ; Microsoft l’avait annoncée quelques semaines auparavant dans un billet de blog dédié.

OWA Light remonte à l’époque d’Exchange 2007 : une interface web volontairement simplifiée, conçue comme solution de repli pour les anciens navigateurs et les connexions lentes, officiellement obsolète depuis août 2024. La justification de sa suppression est liée à la sécurité : un chemin de rendu Legacy séparé en parallèle de l’OWA moderne augmente la complexité et donc la surface d’attaque ; CVE-2026-62914 en apporte la preuve concrète. Les personnes ayant lu l’[article de juillet](/blog/exchange-security-updates-juli-2026) s’en souviendront également : la mitigation CVE-2026-42897 de mai avait déjà rendu OWA Light inutilisable par effet de bord. L’interface était donc déjà condamnée.

### Si vous ne pouvez pas appliquer le correctif : désactiver OWA Light manuellement

Point important pour toutes les personnes qui ne peuvent pas (encore) installer la SU d’août, par exemple faute d’activation ESU : Microsoft recommande expressément de **désactiver manuellement** OWA Light dans ce cas afin d’atténuer CVE-2026-62914. Cela se fait via la stratégie de boîte aux lettres OWA et la page de connexion :

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

La première commande désactive la version Light pour toutes les boîtes aux lettres utilisant la stratégie concernée ; la seconde retire l’option « Utiliser la version Light » de la page de connexion OWA. Les modifications apportées au répertoire virtuel OWA ne prennent effet de manière fiable qu’après le recyclage du pool d’applications OWA ou un `iisreset`.

### Ce que les administrateurs devraient vérifier maintenant

La désactivation est techniquement triviale, mais pas toujours sur le plan organisationnel : OWA Light était la solution de repli discrète pour des scénarios de niche. Il convient désormais de vérifier les favoris et les consignes du helpdesk qui ont `?layout=light` codé en dur, les appareils de kiosque et terminaux équipés d’anciens navigateurs, ainsi que les guides internes destinés aux utilisateurs qui employaient la version Light pour des raisons d’accessibilité. Outlook on the web moderne fonctionne dans tous les navigateurs actuels et intègre ses propres fonctions d’accessibilité ; mais sans information préalable des utilisateurs concernés, les tickets seront inévitables.

## Pourquoi une SU paraît désormais chaque mois et où en est Exchange SE CU1

Deux jours après la publication, l’équipe Exchange a répondu dans un billet de blog remarquablement transparent (« Where is Exchange SE CU1 anyway? ») à la question que se posent de nombreux administrateurs. En bref : Microsoft utilise des outils d’IA à l’échelle du groupe pour identifier des vulnérabilités dans ses propres produits. Les équipes, y compris celle d’Exchange, traitent actuellement les découvertes signalées : validation, reproduction, correction, tests de régression et livraison mensuelle. Depuis mai 2026, une SU Exchange paraît ainsi chaque mois, et Microsoft indique explicitement que ce rythme soutenu se poursuivra.

La très attendue **CU1 pour Exchange SE** est retardée précisément pour cette raison. Initialement annoncée pour le premier semestre 2026, puis reportée au second, elle n’a désormais plus de date cible. Microsoft ne souhaite publier CU1 que lorsqu’un mois sans livraison urgente liée à la sécurité s’intercalera ; une CU immédiatement dépassée par une SU imposerait un double travail de mise à jour à de nombreuses organisations. D’ici là, la charge de sécurité mensuelle est continuellement intégrée au build interne de CU1.

Dans la pratique, cela signifie deux choses. Premièrement : attendre CU1 n’est pas une stratégie, ni pour la migration vers SE ni pour l’installation des SU. Deuxièmement : une **fenêtre de maintenance mensuelle** pour Exchange doit désormais faire partie intégrante du calendrier d’exploitation, comme c’est depuis longtemps une évidence pour les serveurs Windows.

## Installation et suivi

La procédure reste éprouvée : commencez par inventorier, avec l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), les serveurs concernés, leurs niveaux CU/SU et les éventuelles étapes manuelles en attente. Installez ensuite la SU (en cas de CU obsolète, l’[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) indique le chemin), redémarrez le serveur et vérifiez que tous les services Exchange ont démarré correctement. Si des services sont *désactivés*, l’installation a été interrompue ; utilisez alors la solution de contournement documentée dans l’article de support Microsoft consacré à l’erreur de version de fichier ou le [script SetupAssist](https://aka.ms/ExSetupAssist). Enfin, exécutez à nouveau le Health Checker.

Les SU sont cumulatives : les personnes ayant ignoré la SU de juillet peuvent installer directement celle d’août. Et pour les environnements hybrides, le complément habituel s’applique : si le certificat Auth est remplacé après l’installation de la SU, le Hybrid Configuration Wizard doit être exécuté à nouveau.

Un travail de suivi de juillet reste d’actualité : les personnes ayant encore activé la mitigation CVE-2026-42897 (M2.1.0) devraient maintenant la supprimer ; la procédure correcte est expliquée dans l’[article sur la SU de juillet](/blog/exchange-security-updates-juli-2026).

## Procédure recommandée

En résumé : installez rapidement la SU d’août sur tous les serveurs Exchange et les machines équipées des Management Tools : la faille Pwn2Own et la RCE à 8.8 sont des raisons suffisantes pour ne pas attendre le prochain Patch Tuesday. Si vous ne pouvez pas appliquer le correctif immédiatement : OWA Light peut être désactivé manuellement comme mesure immédiate contre CVE-2026-62914. Avant la désactivation d’OWA Light, identifiez et informez les groupes d’utilisateurs concernés (anciens favoris, navigateurs de kiosque, workflows d’accessibilité). Exécutez ensuite le Health Checker, effectuez les travaux de suivi de juillet encore ouverts et prévoyez une fenêtre de maintenance Exchange mensuelle, car ce rythme va perdurer.

## Sources

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Annonce officielle de publication avec les versions prises en charge, l’information sur OWA Light, les problèmes connus et la FAQ ; les commentaires confirment le correctif Pwn2Own (CVE-2026-62911) et la persistance du SettingOverride pour les messages wrapper.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Annonce préalable de la désactivation et recommandation de Microsoft de désactiver manuellement OWA Light en l’absence de correctif.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): L’équipe Exchange sur la recherche de vulnérabilités assistée par IA, le rythme mensuel durable des SU et le retard de CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Référence pour les numéros de build des SU d’août.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Conditions et durée (mai à octobre 2026) du programme ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Problème hybride connu depuis juin et solution de contournement via SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Présentation en allemand des sept CVE avec leurs valeurs CVSS et builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Paramètre `OWALightEnabled` pour désactiver manuellement la version Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventaire des niveaux CU/SU et des étapes manuelles restantes avant et après l’installation.
