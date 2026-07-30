---
title: "SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration"
navTitle: "SEPPmail 15.0.6"
description: "SEPPmail a publié en juillet 2026 la version corrective 15.0.6 et le correctif urgent 15.0.6.1. Outre les vulnérabilités corrigées dans la génération de PDF et le traitement PGP, ces versions apportent un champ MFA distinct, l’authentification LDAP pour l’interface graphique d’administration et des corrections pour RuleEngine, Webmail et l’API REST."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "5 min de lecture"
themen:
  - seppmail
slug: "seppmail-15-0-6-et-15-0-6-1-correctifs-de-securite-et-nouvelles-fonctions-d-administration"
translationId: "article-3046fc35b259929b"
draft: false
translationOf: seppmail-releases-15-0-6-und-15-0-6-1
url: https://rafaelpfister.ch/fr/blog/seppmail-15-0-6-et-15-0-6-1-correctifs-de-securite-et-nouvelles-fonctions-d-administration
translationSourceHash: 5cf19b84bb90403b0a7e2795222b8f853c29c3fe562429df8538e703e565217a
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:49:25.513Z
translationReview: automatic
---

# SEPPmail 15.0.6 et 15.0.6.1 : correctifs de sécurité et nouvelles fonctions d’administration

SEPPmail a publié la version corrective 15.0.6 le 21 juillet 2026, puis le correctif urgent 15.0.6.1 un jour plus tard. La version corrective comble plusieurs vulnérabilités, met à jour OpenSSH et OpenSSL et apporte des améliorations notables pour l’administration. Le correctif urgent corrige deux erreurs dans la RuleEngine, introduites ou devenues visibles avec la version 15.0.6. Les modifications concernent également les appliances exploitées comme HIN Mailgateway, car elles reposent sur le même firmware SEPPmail.

## Correctif urgent 15.0.6.1 du 22 juillet 2026

Le correctif urgent résout deux points dans la RuleEngine. Premièrement, une valeur non définie dans l’objet Message empêchait l’écriture des entrées de journal dans le journal de messagerie. Les messages concernés traversaient donc le système sans être consignés. Deuxièmement, la RuleEngine reconnaît désormais le sens des e-mails archivés afin que leur distribution soit correctement traitée.

Toute personne ayant déjà installé la version 15.0.6 ou prévoyant la mise à jour devrait passer directement à la version 15.0.6.1.

Les appliances HIN semblent également avoir reçu le correctif urgent : un HIN Mailgateway avec la version installée 15.0.6-RC-42-g278c81f84 indique désormais 15.0.6-RC-88-g916e513cc comme prochaine version dans la branche 15.0. Les désignations RC du firmware HIN ne peuvent pas être associées directement à une version SEPPmail, mais le moment de la proposition plaide en faveur du correctif urgent.

## Correctifs de sécurité dans la version 15.0.6

La partie la plus importante de la version corrective est constituée de trois corrections de l’architecture de sécurité :

- Une possible vulnérabilité de traversée de chemin dans la génération de PDF a été corrigée. Elle a été découverte par InfoGuard.
- Tout contenu déchiffré via PGP est désormais encodé en Base64 afin d’empêcher les injections de structure MIME.
- La fonction hashencrypt a été migrée vers AES-256-CBC avec PBKDF2.

S’y ajoutent des bibliothèques mises à jour : OpenSSH 10.4 et OpenSSL 3.0.21 corrigent ensemble plus de vingt CVE. Rien que pour ces points, la mise à jour est recommandée pour les systèmes de production.

## Nouvelles fonctions pour l’administration

Trois modifications de l’interface graphique d’administration se remarquent au quotidien :

- **Champ de saisie MFA distinct :** Le second facteur ne doit plus être ajouté au mot de passe, mais dispose de son propre champ. Cela élimine un piège de longue date lors de la connexion.
- **Authentification LDAP pour l’interface graphique d’administration :** Les administrateurs peuvent désormais s’authentifier auprès d’un serveur LDAP externe au lieu de gérer des comptes locaux sur l’appliance. La configuration est décrite dans l’article sur la [connexion de l’interface graphique d’administration à Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung) décrite. Je teste encore si le HIN Mailgateway a également reçu cette fonction et compléterai l’article ensuite ; comme HIN utilise la même base de firmware, je pars du principe que oui.
- **Bouton AutoRenew pour MPKI :** Dans les paramètres du connecteur MPKI, le renouvellement automatique des certificats peut être déclenché manuellement via « Trigger AutoRenew... ».

Par ailleurs, l’appliance utilise désormais systématiquement des fuseaux horaires valides (standard : Europe/Zurich), et l’ID d’objet système sous System >> Advanced View est validé comme OID valide.

## Traitement des e-mails et Webmail

Quatre points ont été corrigés dans la RuleEngine. Le traitement de l’objet fonctionne désormais également avec un encodage inconnu. Les messages sont rejetés lorsqu’une signature est explicitement demandée, mais ne peut pas être créée ; auparavant, de tels messages pouvaient continuer sans signature. Les copies d’archive passent désormais par la fonction de distribution et reçoivent ainsi des en-têtes ARC. Enfin, pour les messages PGP sans données MDC, les erreurs MDC sont ignorées au lieu de perturber le traitement.

Quatre erreurs ont été corrigées dans le Webmail (GINA) : la suppression automatique des comptes non enregistrés après expiration du délai de grâce fonctionne à nouveau, la fonction hashdecrypt renvoyait dans certains cas un résultat de déchiffrement faussement positif, l’ajout d’une pièce jointe vidait les champs À et CC, et l’affichage de l’heure dans les journaux SMS était erroné.

## API REST, cluster et sauvegarde

L’API REST reçoit des corrections sur plusieurs points de terminaison : /system/ifaliasconfig (gestion des valeurs null), /system/applySysconfig (configuration des accès), /crypto/domain/{domainName} (téléversement de certificats de domaine) ainsi que GET et POST /ssl/csr. Le délai d’attente des appels REST a été augmenté de 300 à 900 secondes, ce qui rend plus fiables les requêtes de longue durée, telles que les modifications importantes de configuration.

En fonctionnement en cluster, une adresse IP CARP existante bloquait jusqu’à présent les réglages IP d’un membre nouvellement ajouté ; ce problème est corrigé. Avant la création quotidienne d’un instantané, la sauvegarde vérifie désormais également si la base de données est corrompue avant d’écrire l’instantané.

## Lien avec la panne de connexion sous la version 15.0.5

Lors de la mise à jour d’un cluster vers la version 15.0.5, la connexion pouvait échouer sur les deux nœuds. Le problème et la procédure de récupération sont décrits dans l’article sur la [panne de connexion après la mise à jour 15.0.5](/blog/hin-update-issue-version-15.0.5) décrit. Le fabricant connaissait déjà le problème à l’époque et avait annoncé une correction dans une version ultérieure.

Les notes de version de la 15.0.6 contiennent désormais précisément une entrée correspondant à ce problème : « prevent password rehashing when cluster members use different firmware versions ». Lors d’une mise à jour de cluster, les nœuds fonctionnent inévitablement temporairement avec des versions de firmware différentes. Si un nœud recalcule les hachages de mots de passe durant cette phase et les réplique dans le cluster, les hachages ne correspondent plus à l’autre version, et la connexion échoue sur les deux nœuds, exactement comme lors de la panne observée à l’époque. Les notes de version ne mentionnent pas explicitement la panne de connexion, mais cette entrée couvre exactement la configuration qui l’avait déclenchée. La cause est donc traitée dans la version 15.0.6 ; la procédure d’urgence avec dissolution du cluster nécessaire sous la 15.0.5 devrait devenir inutile lors de futures mises à jour.

## Corrections mineures

Dans le journal de messagerie, le tri par date a été corrigé : il était jusqu’à présent alphabétique au lieu de chronologique. La taille affichée des messages LFT est de nouveau correcte. Les accès à des en-têtes X inexistants ne sont plus consignés. Le connecteur CertCentral de la MPKI gère plus robustement les erreurs de saisie et REST.

## Évaluation

Les deux erreurs de RuleEngine corrigées par le correctif urgent plaident en faveur de sauter la version 15.0.6 et d’utiliser directement la 15.0.6.1. Pour les clusters, créez des instantanés des deux nœuds avant la mise à jour et respectez l’ordre de mise à jour indiqué dans la documentation du fabricant. La panne de connexion sous la version 15.0.5 a montré pourquoi cette préparation n’est pas une simple formalité.

## Sources

1.  [Documentation SEPPmail – « Revision History »](https://docs.seppmail.com/ch/20_revision-history.html): Notes de version officielles pour les versions 15.0.6 et 15.0.6.1 avec tous les détails.

2.  [HIN Mailgateway 15.0.5 : résoudre la panne de connexion après la mise à jour du cluster](/blog/hin-update-issue-version-15.0.5): Pourquoi les instantanés et le bon ordre de mise à jour sont essentiels dans un cluster.
