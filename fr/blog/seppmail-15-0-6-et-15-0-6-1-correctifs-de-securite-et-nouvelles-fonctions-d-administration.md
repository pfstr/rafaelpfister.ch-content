---
title: "SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration"
navTitle: "SEPPmail 15.0.6"
description: "En juillet 2026, SEPPmail a publié la version corrective 15.0.6 et le correctif urgent 15.0.6.1. Outre des vulnérabilités corrigées dans la génération de PDF et le traitement PGP, ces versions apportent un champ MFA distinct, l’authentification LDAP pour l’interface d’administration et des corrections pour RuleEngine, Webmail et l’API REST."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min de lecture"
themen:
  - seppmail
slug: "seppmail-15-0-6-et-15-0-6-1-correctifs-de-securite-et-nouvelles-fonctions-d-administration"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
translationSourceHash: 636a7246234584a2b5797f53239fe65129de0f4463b8f773d0a7d9ed06d61f91
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:14:53.220Z
translationReview: automatic
url: https://rafaelpfister.ch/fr/blog/seppmail-15-0-6-et-15-0-6-1-correctifs-de-securite-et-nouvelles-fonctions-d-administration
---

# SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration

SEPPmail a publié la version corrective 15.0.6 le 21 juillet 2026, puis le correctif urgent 15.0.6.1 un jour plus tard. La version corrective comble plusieurs vulnérabilités, met à jour OpenSSH et OpenSSL et apporte des améliorations notables pour l’administration. Le correctif urgent corrige deux erreurs dans la RuleEngine, introduites ou rendues visibles avec la version 15.0.6. Les modifications concernent également les appliances exploitées comme HIN Mailgateway, car elles reposent sur le même firmware SEPPmail.

## Correctif urgent 15.0.6.1 du 22 juillet 2026

Le correctif urgent résout deux problèmes dans la RuleEngine. Premièrement, une valeur indéfinie dans l’objet Message empêchait l’écriture d’entrées de journal dans le journal de messagerie. Les messages concernés traversaient donc le système sans être consignés. Deuxièmement, la RuleEngine reconnaît désormais le sens des e-mails archivés afin que leur distribution soit traitée correctement.

Les personnes ayant déjà installé la version 15.0.6 ou prévoyant la mise à jour devraient passer directement à la version 15.0.6.1.

Les appliances HIN semblent également avoir reçu le correctif urgent : un HIN Mailgateway avec la version installée 15.0.6-RC-42-g278c81f84 indique désormais 15.0.6-RC-88-g916e513cc comme prochaine version dans la branche 15.0. Les désignations RC du firmware HIN ne peuvent pas être directement associées à une version SEPPmail, mais le moment de la mise à disposition plaide en faveur du correctif urgent.

## Correctifs de sécurité dans la version 15.0.6

La partie la plus importante de cette version corrective est constituée de trois améliorations de l’architecture de sécurité :

- Une possible vulnérabilité de traversée de répertoires dans la génération de PDF a été corrigée. Elle a été découverte par InfoGuard.
- Tout le contenu déchiffré via PGP est désormais encodé en Base64 afin d’empêcher les injections de structure MIME.
- La fonction hashencrypt a été migrée vers AES-256-CBC avec PBKDF2.

S’y ajoutent des bibliothèques mises à jour : OpenSSH 10.4 et OpenSSL 3.0.21 corrigent ensemble plus de vingt CVE. Ces seuls points rendent la mise à jour recommandable pour les systèmes de production.

## Nouvelles fonctions pour l’administration

Trois changements dans l’interface d’administration se remarquent au quotidien :

- **Champ de saisie MFA distinct :** Le second facteur ne doit plus être ajouté au mot de passe, mais dispose de son propre champ. Cela élimine une source d’erreurs de longue date lors de la connexion.
- **Authentification LDAP pour l’interface d’administration :** Les administrateurs peuvent désormais s’authentifier auprès d’un serveur LDAP externe, au lieu de gérer des comptes locaux sur l’appliance. La configuration est décrite dans l’article sur la [connexion de l’interface d’administration à Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung) . Je teste encore si HIN Mailgateway a également reçu cette fonction et compléterai ensuite l’article ; HIN utilisant la même base de firmware, je le suppose.
- **Bouton AutoRenew pour MPKI :** Dans les paramètres du connecteur MPKI, le renouvellement automatique des certificats peut être déclenché manuellement via « Trigger AutoRenew... ».

Par ailleurs, l’appliance utilise désormais systématiquement des fuseaux horaires valides (par défaut : Europe/Zurich), et l’ID d’objet système sous System >> Advanced View est validé en tant qu’OID valide.

## Traitement des e-mails et Webmail

Quatre points ont été corrigés dans la RuleEngine. Le traitement de l’objet fonctionne désormais aussi avec un encodage inconnu. Les messages sont renvoyés lorsqu’une signature est explicitement demandée mais ne peut pas être créée ; auparavant, ces messages pouvaient continuer sans signature. Les copies archivées passent désormais par la fonction de distribution et reçoivent ainsi des en-têtes ARC. Enfin, pour les messages PGP sans données MDC, les erreurs MDC sont ignorées au lieu de perturber le traitement.

Dans Webmail (GINA), quatre erreurs ont été corrigées : la suppression automatique des comptes non enregistrés après expiration du délai de grâce fonctionne à nouveau, la fonction hashdecrypt renvoyait dans certains cas un résultat de déchiffrement faussement positif, l’ajout d’une pièce jointe vidait les champs À et CC, et l’affichage de l’heure dans les journaux SMS était erroné.

## API REST, cluster et sauvegarde

L’API REST reçoit des corrections sur plusieurs points de terminaison : /system/ifaliasconfig (gestion des valeurs null), /system/applySysconfig (configuration des accès), /crypto/domain/{domainName} (téléversement de certificats de domaine), ainsi que GET et POST /ssl/csr. Le délai d’expiration des appels REST est passé de 300 à 900 secondes, ce qui rend les requêtes de longue durée, telles que les modifications de configuration importantes, plus fiables.

En fonctionnement en cluster, une IP CARP existante bloquait jusqu’à présent les paramètres IP d’un membre nouvellement ajouté ; ce problème est corrigé. Avant la création quotidienne d’un instantané, la sauvegarde vérifie désormais également si la base de données est corrompue avant d’écrire l’instantané.

## Lien avec la panne de connexion sous la version 15.0.5

Lors de la mise à jour d’un cluster vers la version 15.0.5, la connexion pouvait échouer sur les deux nœuds. Le problème et la procédure de restauration sont décrits dans l’article sur la [panne de connexion après la mise à jour 15.0.5](/blog/hin-update-issue-version-15.0.5) . Le fabricant connaissait déjà le problème à l’époque et avait annoncé une correction dans une version ultérieure.

Les notes de version de la 15.0.6 contiennent désormais exactement une entrée correspondant à ce problème : « prevent password rehashing when cluster members use different firmware versions ». Lors d’une mise à jour de cluster, les nœuds fonctionnent inévitablement temporairement avec des versions de firmware différentes. Si un nœud recalcule les hachages de mots de passe durant cette phase et les réplique dans le cluster, les hachages ne correspondent plus sur l’autre version et la connexion échoue sur les deux nœuds, exactement comme lors de la panne observée à l’époque. Les notes de version ne mentionnent pas explicitement la panne de connexion, mais l’entrée couvre exactement la configuration qui l’avait provoquée. La cause est donc traitée dans la version 15.0.6 ; la procédure d’urgence nécessaire avec la version 15.0.5, impliquant la dissolution du cluster, devrait devenir superflue lors de futures mises à jour.

## Corrections mineures

Dans le journal de messagerie, le tri des dates a été corrigé : il effectuait auparavant un tri alphabétique au lieu d’un tri chronologique. La taille affichée des messages LFT est à nouveau correcte. Les accès à des en-têtes X inexistants ne sont plus consignés. Le connecteur CertCentral de MPKI gère de manière plus robuste les erreurs de saisie et REST.

## Évaluation

Les deux erreurs de RuleEngine corrigées par le correctif urgent incitent à ignorer la version 15.0.6 et à déployer directement la version 15.0.6.1. Pour les clusters, créez des instantanés des deux nœuds avant la mise à jour et respectez l’ordre de mise à jour indiqué dans la documentation du fabricant. La panne de connexion sous la version 15.0.5 a montré pourquoi cette préparation n’est pas un simple formalisme.

## Sources

1.  [Documentation SEPPmail – « Revision History »](https://docs.seppmail.com/ch/20_revision-history.html): Notes de version officielles de 15.0.6 et 15.0.6.1 avec tous les points détaillés.

2.  [HIN Mailgateway 15.0.5 : corriger la panne de connexion après la mise à jour du cluster](/blog/hin-update-issue-version-15.0.5): Pourquoi les instantanés et le bon ordre de mise à jour sont essentiels dans un cluster.
