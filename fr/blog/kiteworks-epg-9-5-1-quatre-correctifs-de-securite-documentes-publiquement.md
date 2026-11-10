---
title: "Kiteworks EPG 9.5.1 : quatre correctifs de sécurité documentés publiquement"
navTitle: "Kiteworks EPG 9.5.1"
description: "Kiteworks Email Protection Gateway 9.5.1 corrige quatre vulnérabilités documentées publiquement, dont une prise de contrôle de compte avec un CVSS de 9,4. Les installations antérieures à la version 9.5.1 doivent être mises à jour vers cette version ou une version ultérieure."
date: "2026-11-06"
kategorie: "Totemomail"
timeToRead: "3 min de lecture"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "releases"
slug: "kiteworks-epg-9-5-1-quatre-correctifs-de-securite-documentes-publiquement"
translationId: article-d4b9d984c72a54ea
translationOf: kiteworks-epg-releases-9-5-1
url: https://rafaelpfister.ch/fr/blog/kiteworks-epg-9-5-1-quatre-correctifs-de-securite-documentes-publiquement
translationSourceHash: 7d961f479006e9eb0da1a74925d77e25e7205a64a1ed783dccb55b6eed8c2c5c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:19:26.330Z
translationReview: required
---

# Kiteworks EPG 9.5.1 : quatre correctifs de sécurité documentés publiquement

Kiteworks poursuit l’ancienne technologie totemo sous le nom d’Email Protection Gateway (EPG). La version 9.5.1 était au plus tard le 25 septembre 2026 la version actuelle de Kiteworks ; le fabricant ne mentionne pas de date de publication publique. Les avis du fabricant publiés le 30 septembre documentent quatre correctifs pour EPG 9.5.1 : une prise de contrôle de compte, deux vulnérabilités non authentifiées permettant respectivement l’écriture de fichiers et le déclenchement d’un déni de service, ainsi qu’une vulnérabilité permettant aux administrateurs d’écrire des fichiers arbitraires. La classification pour la version totemomail du même nom figure dans l’article sur [totemomail 9.5.1](/blog/totemomail-9-5-1-sicherheitsupdate).

## Aperçu des versions

| Version | Date | Type | Points essentiels |
|---|---|---|---|
| 9.5.1 | confirmée publiquement au plus tard le 25.09.2026 | mise à jour de sécurité | Quatre correctifs EPG documentés publiquement, dont CVE-2026-102149 avec un CVSS de 9,4 |

## Correctifs de sécurité dans la version 9.5.1

Selon Kiteworks, les quatre avis concernent toutes les versions d’EPG antérieures à la version 9.5.1 et indiquent la version 9.5.1 comme version corrigée. Les avis ont été publiés le 30 septembre 2026. Ils décrivent rétrospectivement l’état de sécurité de la version, et non sa date de publication exacte.

- **CVE-2026-102149, CVSS 9,4 (critique) :** Une faiblesse dans le contrôle d’accès pouvait permettre à un attaquant d’accéder à des comptes utilisateurs via le réseau. L’évaluation CVSS ne requiert ni privilèges ni interaction de l’utilisateur. Kiteworks attribue la cause à CWE-306 (absence d’authentification pour des fonctions critiques) et CWE-639 (contournement de l’autorisation via une clé contrôlée par l’utilisateur).
- **CVE-2026-102143, CVSS 7,5 (élevé) :** Un attaquant non authentifié pouvait écrire des fichiers sur l’appliance. Kiteworks mentionne un problème d’authentification ainsi que CWE-306 et CWE-434 (téléversement non restreint de types de fichiers dangereux).
- **CVE-2026-102146, CVSS 6,5 (moyen) :** Un administrateur pouvait écrire des fichiers arbitraires. Kiteworks décrit une erreur de privilèges ; la métrique CVSS requiert des privilèges élevés. Le fabricant cite comme faiblesses CWE-73 (contrôle externe du nom de fichier ou du chemin) et CWE-1336 (neutralisation insuffisante dans un moteur de modèles).
- **CVE-2026-102144, CVSS 5,3 (moyen) :** Un attaquant sur le réseau pouvait déclencher un déni de service. L’avis mentionne l’absence d’authentification pour des fonctions critiques et une consommation non contrôlée de ressources (CWE-306, CWE-400). Cette évaluation ne requiert pas non plus de privilèges ni d’interaction de l’utilisateur.

Pour les exploitants, la prise de contrôle de compte est le correctif prioritaire, car elle est, selon l’avis, possible sans compte existant. Les deux autres vulnérabilités sans connexion concernent également toutes les versions antérieures à la version 9.5.1. Les avis du fabricant ne publient aucun détail technique sur les points de terminaison, les configurations ou les solutions de contournement.

## Procédure recommandée

Mettez EPG à niveau au minimum vers la version 9.5.1 si une version antérieure est utilisée. Pour les quatre avis, Kiteworks indique explicitement la mise à niveau vers la version 9.5.1 ou une version ultérieure comme mesure corrective. Si une version ultérieure du fabricant est déjà validée, elle satisfait également à cette version minimale. Kiteworks n’a pas publié dans les sources consultées de chemin de mise à jour public, de problèmes connus ni de périodes de support pour EPG 9.5.1.

## Sources

1.  [Kiteworks: Precautionary Shutdown Advisory for Customers](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Communication du fabricant du 25 septembre 2026 indiquant la version 9.5.1 comme version actuelle.

2.  [Kiteworks Security Advisories](https://github.com/kiteworks/security-advisories/security/advisories): Aperçu officiel des avis publiés le 30 septembre 2026, dont les entrées EPG pour la version 9.5.1.

3.  [GHSA-c9w5-4frw-7wqq: Account takeover (CVE-2026-102149)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-c9w5-4frw-7wqq): Faiblesse critique de contrôle d’accès, CVSS 9,4, corrigée dans EPG 9.5.1.

4.  [GHSA-3p9g-jh62-8f89: Unauthorized file modification (CVE-2026-102143)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-3p9g-jh62-8f89): Écriture de fichiers non authentifiée, CVSS 7,5, corrigée dans EPG 9.5.1.

5.  [GHSA-5pgq-v8g2-rg2f: Unauthorized file modification (CVE-2026-102146)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5pgq-v8g2-rg2f): Écriture de fichiers arbitraires avec des droits d’administrateur, CVSS 6,5, corrigée dans EPG 9.5.1.

6.  [GHSA-wwhf-5862-rjxq: Denial of service (CVE-2026-102144)](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-wwhf-5862-rjxq): Déni de service non authentifié, CVSS 5,3, corrigé dans EPG 9.5.1.

7.  [Kiteworks: EPG with totemo Integration](https://www.kiteworks.com/video-why-kiteworks-and-totemo-are-better-together/): Indication du fabricant concernant la poursuite et le changement de nom de la technologie totemo en Email Protection Gateway.
