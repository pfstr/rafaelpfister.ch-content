---
title: "Totemomail et Kiteworks EPG : passerelle de messagerie et changement de plateforme"
blatt: "totemomail"
description: "Totemomail et son successeur Kiteworks Email Protection Gateway pour les administrateurs de messagerie : architecture SMTP inline, politiques, S/MIME, OpenPGP et TLS, WebMail, pile cryptographique Java, état des clés, identité, licences, migration, haute disponibilité, supervision et reprise."
fakten:
  - label: Gamme de produits
    wert: totemomail → Kiteworks Email Protection Gateway
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Rôle système
    wert: passerelle de chiffrement des e-mails pilotée par des politiques
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Transport
    wert: SMTP dans le chemin des messages
    href: https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview
  - label: Procédés
    wert: S/MIME · OpenPGP · TLS
    href: https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf
  - label: Sans clé du destinataire
    wert: distribution via WebMail ou portail
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Décision
    wert: politiques centralisées et sélection automatique des clés
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Cœur cryptographique historique
    wert: Totemo Security Platform · TCM basé sur Java
    href: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500
  - label: Plateforme EPG
    wert: Kiteworks Hardened Virtual Appliance
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Stockage des données EPG
    wert: base de données intégrée, non administrée directement par le client
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Observabilité
    wert: preuve du flux de messagerie · Syslog · audit de plateforme
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Modes d’exploitation EPG
    wert: on-premises · ressources cloud propres · hébergement mono-locataire
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Origine
    wert: totemo AG, Suisse · fondée en 2001 · acquise par Kiteworks en 2022
    href: https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - totemomail
translationSourceHash: d949453b9a80abb64f76a8ecdee768813548ae2987b18b071cc23d33b0c5df21
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:53:05.150Z
translationReview: automatic
---

# Totemomail et Kiteworks EPG : passerelle de messagerie et changement de plateforme

Totemomail était une famille de produits de la société suisse totemo AG destinée au chiffrement centralisé des e-mails. Après son acquisition par Kiteworks, le code a, selon le fabricant, été intégré à la plateforme Kiteworks et poursuivi sous le nom **Email Protection Gateway**, EPG. Les noms ne désignent donc ni deux produits totalement indépendants, ni le même environnement d’exploitation : une appliance Totemomail existante et un EPG au sein de la Kiteworks Hardened Virtual Appliance ont des limites différentes en matière de plateforme, de base de données, d’administration et de reprise ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Sur le plan fonctionnel, le rôle central reste celui d’une passerelle de messagerie : elle traite les e-mails selon des politiques centralisées et utilise S/MIME, OpenPGP ou TLS ; une solution WebMail ou portail est disponible pour les destinataires ne disposant pas de clés appropriées. La documentation du fabricant décrit la gestion automatique des clés et des certificats ainsi que l’utilisation avec des clients de messagerie classiques sans plug-in client spécifique à EPG comme modèle de base ([Kiteworks: Enterprise packaging and EPG features](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf), [Microsoft Marketplace: totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)).

Pour les administrateurs de messagerie, la gamme de produits constitue avant tout une chaîne de frontières de confiance. La passerelle accepte un message via SMTP, classe le sens et les identités, détermine une politique, recherche des clés ou un canal de secours, modifie au besoin le contenu MIME et transmet le message au saut suivant. Avec WebMail s’ajoutent un état propre au Web, aux comptes, aux clés et au stockage. L’affirmation « le message est chiffré » est incomplète sans indication de la méthode, du tronçon protégé, du détenteur de la clé et du moment considéré ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

L’explication suit un message à travers les règles EPG, le choix des clés et les procédés de chiffrement jusqu’à la livraison directe ou WebMail. Le passage de Totemomail à Kiteworks, l’exploitation, le diagnostic et la reprise sont ensuite traités séparément.

Totemomail et Kiteworks EPG se placent en tant que passerelle inline dans le chemin SMTP. Un message est accepté, attribué à une identité et à une politique, protégé par une méthode adéquate, puis fourni via SMTP ou portail. Le changement de plateforme modifie l’implémentation et l’exploitation, mais non cette mission fondamentale.

## Approche architecturale : passerelle inline avec politique centralisée

En fonctionnement de passerelle, le système de messagerie interne envoie les messages sortants par [SMTP](/kb/smtp) à Totemomail ou EPG. Les messages entrants atteignent directement la passerelle via MX ou via un système de sécurité de messagerie en amont. La passerelle termine la session SMTP, traite le message et ouvre une nouvelle session pour le transfert. Une poignée de main TCP ou TLS réussie ne constitue pas encore une acceptation du message ; le transfert de responsabilité n’intervient qu’avec la réponse SMTP positive à la fin de la phase DATA ([RFC 5321, section 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

La description publique du produit EPG distingue plusieurs chemins de protection. « Encryption Gateway » couvre le chiffrement et le déchiffrement externes à la passerelle. « Internal Encryption » protège les messages durant leur transit et leur stockage dans une plateforme de messagerie cloud. Parmi les autres composants documentés figurent Directory Proxy, Application Integration Proxy et un connecteur pour RMS ou Microsoft Information Protection. Les rôles effectivement sous licence et activés doivent être déterminés à partir de votre propre configuration et du contrat ; le seul nom du produit ne les prouve pas ([Kiteworks: Enterprise packaging and licensing](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

Pour Microsoft 365, l’intégration de transport reste un problème de connecteurs. Exchange Online doit router les messages de manière ciblée vers la passerelle et accepter le trafic de retour en fonction d’une identité fiable du pair. Un MX devant une passerelle tierce modifie en outre l’adresse IP source que Exchange Online voit pour l’évaluation du spam et de l’usurpation. Microsoft documente à cet effet les connecteurs et Enhanced Filtering ; les deux sont distincts de la politique cryptographique EPG ([Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

## Pile technologique et limites de produit vérifiables

Pour la plateforme historique Totemomail, le cœur cryptographique est bien documenté publiquement. Les documents NIST-CMVP décrivent le **Totemo Cryptographic Module**, TCM, comme une bibliothèque Java commune de la Totemo Security Platform, TSP. La validation de 2019 mentionne Totemo Appliance OS 2.3, JRE 8 et un module cryptographique logiciel avec AES, RSA, ECDSA, la famille SHA, HMAC et un générateur de nombres aléatoires ; l’entrée a désormais un statut historique et ne constitue donc pas une preuve d’acquisition actuellement admissible ni d’un mode d’algorithme concret installé ([NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500), [Totemo TCM Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)).

La politique de sécurité NIST délimite expressément TCM comme un composant de TSP. Elle ne documente ni un flux de messagerie Totemomail complet ni une implémentation universelle de file d’attente, de base de données ou de MTA. Ces détails ne doivent donc pas être transposés à toutes les versions de Totemomail ou d’EPG à partir d’une installation Apache James étrangère ou d’un constat isolé sur le terrain. Pour un document d’exploitation, la configuration produit exportée, la matrice de support, l’inventaire des paquets ou services et le manuel du fabricant de la génération réellement installée font foi.

Avec EPG, la limite se déplace vers la Kiteworks Hardened Virtual Appliance. La FAQ de migration indique que le système d’exploitation, le serveur Web et la base de données sont inclus dans l’appliance et que les administrateurs clients n’ont pas d’accès direct au système d’exploitation ou à la base de données. Les mises à jour sont appliquées sous la forme d’une unité d’appliance testée. Cela réduit l’exploitation des composants côté client, mais fait des procédures de sauvegarde, de cluster, de mise à jour et de support de l’appliance la principale surface de contrôle ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Espace d’état | Totemomail historique | Kiteworks EPG | Preuve d’administration |
|---|---|---|---|
| SMTP et routage | configuration de la passerelle, du MTA et de la destination de l’installation | EPG plus modèle réseau et rôles Kiteworks | e-mail de test, réponse SMTP, ID de file d’attente/de flux de messagerie, saut suivant |
| Politique | règles et déclencheurs Totemomail | politique EPG intégrée plus contrôles de plateforme | politique exportée, priorité, état activé, matrice de test |
| Clés | TSP/TCM, certificats et clés OpenPGP | stock de clés et certificats EPG migré | empreinte, détenteur, expiration, historique, test de déchiffrement |
| Livraison Web | Totemomail WebMail et comptes destinataires | portail EPG au sein de la plateforme Kiteworks | inscription, connexion, conservation, ouverture d’un ancien message |
| Base de données | base de données externe ou locale selon l’installation | base de données intégrée sans accès administrateur direct | tâche de sauvegarde, procédure de restauration, chemin de support du fabricant |
| Journaux | journaux de produit, de flux de messagerie et système | journaux de plateforme et EPG normalisés, export Syslog | ID de message/événement corrélable, base temporelle, arrivée SIEM |

La comparaison suit la FAQ de migration du fabricant et les descriptions publiées des fonctions EPG. Lorsque les détails de Totemomail dépendent de l’installation, cela reste délibérément un point à vérifier plutôt qu’une propriété de produit inventée ([Kiteworks: totemomail-to-EPG migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: Top reasons for EPG](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-totemomail.svg?v=20260813" title="Interaktive Infografik: Totemomail- und Kiteworks-EPG-Mailflow, Policy, Schlüssel, WebMail, Plattformmigration, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-totemomail.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Flux de messagerie SMTP et risque de boucle

Un chemin de passerelle requiert une responsabilité univoque dans chaque sens. En sortie, le système de messagerie ne doit pas pouvoir livrer tantôt directement, tantôt via EPG, lorsque seul le chemin de passerelle applique la politique prescrite. En entrée, il doit être établi si MX, les filtres en amont ou EPG authentifient l’expéditeur d’origine et quel saut atteint le serveur de messagerie interne. Plusieurs connecteurs bidirectionnels sans autorité claire sur les destinataires et les domaines peuvent faire circuler les messages en boucle entre les systèmes ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3), [Microsoft: Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Pour chaque route de production, les administrateurs devraient documenter ensemble les valeurs suivantes :

- réseau source ou identité de pair basée sur certificat et nom EHLO attendu ;
- adresse d’acceptation locale, port, exigences STARTTLS et certificat client ;
- responsabilité de domaine autoritatif, de relais ou de domaine fractionné ;
- sens de la politique, classification des utilisateurs et des destinataires ;
- saut suivant, résolution DNS ou smarthost et exigence TLS ;
- comportement en cas d’indisponibilité d’un composant de politique, de clé, d’annuaire ou de portail ;
- nombre maximal de sauts, âge de la file d’attente, responsabilité des nouvelles tentatives et des rebonds.

La norme SMTP impose d’ajouter des lignes Received à chaque transfert et décrit la détection de boucles à partir du chemin de trace. Pour le diagnostic et l’audit, l’ID d’événement de la passerelle doit pouvoir être corrélé à Internet `Message-ID`, à l’enveloppe, à la fenêtre temporelle et à la trace en aval. Un objet visible n’est pas un identifiant stable et peut être modifié par une politique ou la livraison Web ([RFC 5321, section 4.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.4)).

Après l’acceptation SMTP, le moteur de politiques décide, selon l’expéditeur, le destinataire, l’annuaire et les clés disponibles, quel chemin de protection est autorisé et possible.

## Moteur de politiques et décision d’identité

Kiteworks décrit EPG comme une couche de décision automatique pilotée par des politiques. Selon l’étendue des fonctions sous licence, les règles peuvent reposer sur les domaines, les utilisateurs, les emplacements, les caractéristiques de contenu ou les étiquettes Microsoft Information Protection. L’automatisation n’élimine toutefois pas la nécessité de faire confiance aux entrées : un en-tête d’expéditeur, un déclencheur dans l’objet ou une étiquette définie de l’extérieur ne doivent pas avoir sans contrôle le même effet qu’une identité interne authentifiée ([Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: totemo and Kiteworks better together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)).

Un modèle de règles robuste distingue quatre questions :

1. **Sens :** interne vers externe, externe vers interne, interne vers interne ou application vers externe.
2. **Identité :** l’enveloppe SMTP, le `From:` visible, la session authentifiée, le connecteur, l’objet d’annuaire et le tenant ne sont pas la même chose.
3. **Action :** signer, chiffrer, rechiffrer, livrer au portail, transmettre sans modification, rejeter temporairement ou générer un rebond permanent.
4. **Chemin d’erreur :** clé manquante, chaîne non valide, annuaire indisponible, erreur de politique et saut suivant indisponible nécessitent des résultats distincts.

Avant toute modification, une petite matrice de test comportant exactement un expéditeur et un destinataire par classe d’identité devrait être exécutée. Pour chaque ligne sont consignés la règle attendue, la méthode, l’expéditeur visible, le canal du destinataire, l’indicateur de journal et le comportement en cas d’erreur. Ce n’est que lorsque ces résultats sont reproductibles que la politique est non seulement configurée, mais aussi démontrée.

## Procédés de protection et rechiffrement

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) utilise CMS et des certificats X.509. Pour la signature ou le déchiffrement, la passerelle a besoin d’une clé privée et, pour le chiffrement destiné au destinataire, d’un certificat public approprié. Le chemin de certification, le nom ou l’adresse e-mail, l’usage de clé, la validité et les informations de révocation font partie de la vérification ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utilise un modèle propre de clés, signatures et paquets. L’empreinte, l’ID utilisateur, les capacités, l’expiration et la révocation doivent être inventoriés indépendamment de X.509. Une clé publique apprise automatiquement n’est digne de confiance qu’à hauteur du processus documenté d’apprentissage et de confirmation. La documentation produit mentionne côte à côte S/MIME et OpenPGP, mais n’en fait pas des modèles de confiance interchangeables ([Kiteworks: Enterprise EPG feature matrix](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protège une connexion de transport individuelle. Avec STARTTLS opportuniste, la livraison peut revenir au texte en clair en l’absence de politique supplémentaire. Selon l’environnement, une identité de pair contraignante peut être définie par la validation de certificat, DANE ou MTA-STS ; elle ne protège néanmoins pas le message contre les MTA impliqués ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

Lors du rechiffrement, la passerelle déchiffre un procédé entrant et en crée un nouveau pour le tronçon suivant. Cela permet l’inspection du contenu ou un changement de protocole, mais crée un instant de texte en clair à l’intérieur de la frontière de la passerelle. Le catalogue des fonctions EPG décrit expressément des scénarios où des messages protégés en interne sont brièvement déchiffrés à la passerelle afin d’être inspectés ou convertis vers le standard du destinataire ([Kiteworks: Internal Encryption and Encryption Gateway](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). DLP, antivirus, fichiers temporaires, images mémoire, journaux de débogage et accès administrateur font alors partie de la frontière cryptographique.

En l’absence de clé de destinataire directement utilisable, la livraison au portail déplace l’accès au texte en clair vers une application Web. L’identité Web, la session et le message stocké deviennent ainsi partie intégrante du modèle d’exploitation.

## WebMail et livraison par portail

La FAQ officielle de migration confirme qu’EPG fournit une fonction équivalente à Totemomail WebMail pour les destinataires sans chiffrement e-mail adapté. Les guides publics d’opérateurs pour Totemomail montrent le flux utilisateur typique : notification, premier mot de passe ou lien d’inscription, changement de mot de passe, connexion au portail et choix entre WebMail, S/MIME ou OpenPGP ([Kiteworks: Migration FAQ, question 7](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [L-Bank: Communication e-mail sécurisée](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)).

La livraison par portail constitue un chemin distinct de stockage et d’identité. L’e-mail de notification ne contient pas nécessairement le message confidentiel ; son contenu peut se trouver sur le portail ou dans une enveloppe protégée par mot de passe. Pour les administrateurs, au minimum le nom d’hôte, le certificat TLS, l’état des comptes, les procédures de mot de passe et de réinitialisation, la durée de conservation, les clés de message, l’historique de livraison et le processus de suppression sont donc pertinents. La variante choisie doit découler de votre propre politique ; les guides externes d’opérateurs prouvent des fonctions Totemomail possibles, mais aucun paramètre par défaut global.

Un contrôle de santé du portail ne doit pas s’arrêter à HTTP 200. Le test synthétique doit inclure l’inscription ou un compte de test existant, la connexion, l’ouverture d’un message connu à l’avance, le téléchargement d’une pièce jointe, la réponse et la déconnexion. Après une restauration, il convient en particulier de vérifier si les anciens contenus du portail restent lisibles avec leurs clés historiques.

## Annuaire et clés avec comptage des licences

Kiteworks documente l’intégration LDAP/AD, les rôles et les politiques centralisées au niveau de la plateforme. Pour une passerelle, l’annuaire peut classer les expéditeurs et destinataires, fournir des conditions de groupe et associer les identités à des certificats ou des clés. Dès qu’une règle dépend d’une requête d’annuaire, le point de terminaison [LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen), la base de recherche, le filtre, le compte de liaison, la confiance TLS, le délai d’attente, le cache et le basculement font partie du chemin de message ([Kiteworks: security and integration overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511), [RFC 4513](https://datatracker.ietf.org/doc/html/rfc4513)).

Le stock de clés n’est pas un sous-produit de l’importation des utilisateurs. Il faut inventorier les clés privées d’utilisateurs et de domaines, les clés publiques de destinataires, les certificats CA et intermédiaires, l’état de confiance, les informations de révocation, les connexions aux serveurs de clés ou MPKI ainsi que l’association aux messages historiques. Un nettoyage d’annuaire ne doit jamais supprimer sans vérification une clé privée nécessaire aux anciens messages.

Un document de packaging Kiteworks publié définissait les sièges EPG de telle sorte qu’un utilisateur interne consommait un siège en envoyant ou en recevant un message chiffré, ou en envoyant un message signé ; les utilisateurs internes traversant le système sans chiffrement ni signature y étaient traités différemment. Le document est une description contractuelle datée, non une garantie de licence intemporelle ([Kiteworks: Enterprise 8.0 licensing and packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). Pour l’exploitation, trois valeurs doivent donc être suivies séparément : les identités techniquement connues, les identités réellement actives sur le plan cryptographique et la métrique sous licence contractuelle de votre commande.

Lors du passage à EPG, il faut donc non seulement reprendre ou reconstruire volontairement les routes SMTP, mais aussi les règles, identités, clés, états de portail et preuves.

## Migration de Totemomail vers EPG

Kiteworks ne décrit pas la migration comme un simple renommage in situ. Un transfert automatisé copie les informations de la base de données Totemomail existante vers la base de données EPG. L’ancienne base de données externe doit être décommissionnée après la mise hors service du système Totemomail, car elle n’est plus synchronisée ensuite. EPG lui-même inclut base de données, système d’exploitation et services Web dans la Hardened Virtual Appliance ([Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

Une migration complète comprend au minimum :

- domaines, routes, identités de relais et de connecteurs ainsi que politiques TLS ;
- règles actives, priorités, déclencheurs, exceptions et chemins d’erreur ;
- clés S/MIME, OpenPGP, de domaine, TLS et CA, historique compris ;
- comptes WebMail, messages de portail, conservation, modèles et nom public ;
- intégrations d’annuaire, SSO, SIEM, Syslog, DLP, archivage et applications ;
- rôles, administrateurs et accès d’audit et de support ;
- métrique de licence, parc d’utilisateurs actifs et limites contractuelles ;
- sauvegarde, restauration, HA, DNS, supervision et runbooks.

Le test de réception doit vérifier les données historiques en plus des nouveaux e-mails. Pour chaque ancien type de chiffrement, un message créé avant la migration est ouvert ou déchiffré. Dans chaque sens, un nouvel e-mail de test est envoyé avec et sans clé de destinataire existante. Les cas négatifs, tels qu’un certificat non valide, un saut suivant inaccessible et un annuaire indisponible, sont également vérifiés. Ce n’est qu’ensuite que l’ancienne passerelle peut être retirée de DNS, des connecteurs, des listes d’autorisation et de la supervision.

## Haute disponibilité avec sauvegarde et reprise

La FAQ de migration EPG mentionne le clustering intégré, la sauvegarde/restauration, la reprise après sinistre, la haute disponibilité et la distribution comme propriétés de la Hardened Virtual Appliance. Elle n’indique cependant pas que chaque fonction EPG individuelle reçoit automatiquement le même RPO et RTO. La file d’attente de messagerie, la base de données, les clés, le contenu du portail, le journal d’audit, la configuration et le stockage externe d’objets ou de fichiers doivent être attribués séparément selon l’architecture ([Kiteworks: Migration FAQ – Hardened Virtual Appliance](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Objet de reprise | Preuve minimale | Lacune typique |
|---|---|---|
| Configuration de passerelle | état cible exporté ou sauvegardé et restauration sur système de remplacement | configuration présente, secrets ou certificats manquants |
| Clés privées | liste d’empreintes avant et après restauration ; déchiffrement historique | clés actuelles présentes, ancienne rotation perdue |
| Politique | état activé plus matrice de tests positifs et négatifs | l’export ne correspond pas à la règle réellement active |
| État des e-mails et du portail | objets de test de file d’attente et WebMail connus à l’avance après restauration | la plateforme démarre, mais l’ancien message est illisible |
| Cluster | basculement avec test SMTP, HTTPS, clés et journaux | le VIP bascule, mais pas l’état backend ou l’IP sortante |
| Audit et Syslog | copie externe des événements avec corrélation temporelle et de message | la restauration locale efface l’historique d’enquête |

Pour Totemomail historique, la procédure de sauvegarde dépend de la version et de l’installation. L’affirmation EPG concernant la sauvegarde intégrée ne doit pas être appliquée rétroactivement à une ancienne base de données externe ou à un stockage WebMail séparé. Le chemin de support du fabricant, la documentation installée et un test de restauration isolé constituent ici les sources primaires.

## Supervision et opérations de sécurité

Kiteworks confirme pour EPG l’export vers une ou plusieurs destinations Syslog ainsi que l’intégration Splunk. La documentation de plateforme décrit des journaux et métadonnées normalisés pour l’e-mail et d’autres canaux de communication. Pour les administrateurs de messagerie, la corrélation fonctionnelle reste décisive : l’audit de plateforme, l’événement SMTP, la décision de politique, l’opération de clé, l’accès au portail et la trace de message en aval doivent être réunis selon le temps, l’identité et l’identifiant de message ([Kiteworks: Migration FAQ – syslog](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG logging and audit](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Un tableau de bord administrateur doit au minimum afficher les acceptations et rejets, l’âge des files d’attente par saut suivant, les erreurs de politique, le nombre et le type d’actions cryptographiques, les clés manquantes ou non valides, l’expiration des certificats, la latence LDAP, les erreurs de connexion au portail, l’état des disques et de la base de données, la réplication de cluster, l’âge des sauvegardes et la perte Syslog. Une file d’attente vide n’est un constat positif que si des messages sont simultanément acceptés et transmis avec succès au saut suivant.

Syslog via UDP n’offre aucune livraison garantie. Pour les données d’audit à protéger, il faut prévoir le transport TLS, la mise en tampon, l’authentification de destination, une heure correcte et une alerte de panne ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [RFC 5425](https://datatracker.ietf.org/doc/html/rfc5425)). Les journaux peuvent contenir des expéditeurs, destinataires, objets, informations de certificat ou activités de portail ; la minimisation des données et la protection des accès font donc partie de la politique de journalisation.

Les alertes de sécurité historiques montrent que l’interface Web fait également partie de la surface d’attaque. Des entrées NVD documentent notamment d’anciennes vulnérabilités CSRF, d’accès aux objets et XSS dans des composants Web Totemomail. Il n’en découle aucune conclusion sur un état de correctifs quelconque non vérifié, mais bien l’obligation permanente d’intégrer les avis du fabricant, l’exposition, le statut de support et les tests du portail Web à la gestion des correctifs ([NVD: CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563), [NVD: CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918), [NVD: CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)).

Le diagnostic suit un message de test, de l’acceptation à la sortie SMTP ou l’accès au portail, en passant par la décision de politique et de clé.

## Outils de diagnostic

Le diagnostic commence par le chemin de messagerie publié, puis suit les événements EPG jusqu’à la livraison ou WebMail. La migration de plateforme et l’état des licences ne sont vérifiés qu’une fois l’étape de traitement concernée déterminée.

### DNS et chemins de messagerie publics

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName epg.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.epg.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A epg.example.ch
dig +short AAAA epg.example.ch
dig +short TLSA _25._tcp.epg.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affichent le MX public, l’adresse de la passerelle et, le cas échéant, DANE-TLSA. Les vues des résolveurs internes et externes doivent être vérifiées séparément. Avec Microsoft 365, le MX ne décrit que le premier saut Internet ; les connecteurs et règles de transport déterminent la suite de la route.

### Accessibilité TCP, SMTP et TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection epg.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection epg.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://epg.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz epg.example.ch 25
nc -vz epg.example.ch 443
openssl s_client -starttls smtp -connect epg.example.ch:25 \
  -servername epg.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) ne prouvent que le chemin TCP. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) demande SMTP-STARTTLS ; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affiche les capacités, la poignée de main, SNI et la chaîne de certificats. Seule une transaction SMTP complète avec réponse DATA positive prouve l’acceptation.

### Message de test SMTP autorisé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://epg.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\epg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server epg.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data epg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) et [`swaks`](https://jetmore.org/john/code/swaks/) envoient ici un message de test contrôlé. L’expéditeur, le destinataire et la destination doivent être explicitement autorisés à cette fin. Il faut consigner la réponse SMTP finale, l’ID de passerelle, la politique déclenchée, la clé utilisée ou le chemin de portail, le saut suivant et l’arrivée effective. Un `250` sur `RCPT TO` ne constitue pas encore l’acceptation du contenu du message.

### Chemin WebMail ou portail

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Portaldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest), [`curl`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) vérifient l’interface HTTP et TLS. Un code d’état réussi ne prouve ni la connexion ni l’accès aux clés. Un test de navigateur synthétique doit couvrir l’authentification, l’ouverture d’un message connu, une pièce jointe et la réponse.

### Chemin de paquets au bon point de mesure

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add EPG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name epg.etl
pktmon stop
pktmon pcapng epg.etl -o epg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w epg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ne voient que le trafic au point de mesure choisi. Un client administrateur n’observe pas automatiquement le chemin entre Exchange Online, l’équilibreur de charge et EPG. Pour l’analyse, un hôte autorisé, un port miroir ou des journaux de flux cloud conviennent ; SMTP avant STARTTLS et les métadonnées Web doivent être conservés avec une protection adéquate.

## Histoire technique

Totemo a été fondée en 2001 en Suisse et a développé des solutions de chiffrement des e-mails, de transfert de fichiers géré et de communication mobile. Les validations NIST de 2012 et 2019 documentent l’évolution du Totemo Cryptographic Module basé sur Java et de la Totemo Security Platform sur leurs propres systèmes d’exploitation d’appliance ([Entrust Partner Directory: Totemo](https://www.entrust.com/partners/directory/totemo), [NIST: TCM Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf), [NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)).

Kiteworks a acquis totemo en janvier 2022. La documentation du fabricant décrit l’intégration du code Totemomail dans la plateforme Kiteworks et son renommage en Email Protection Gateway. EPG a associé les fonctions existantes de chiffrement des e-mails à la Hardened Virtual Appliance, à des rôles partagés, des journaux normalisés et d’autres canaux de communication Kiteworks ([Kiteworks: acquisition and EPG integration](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: January 2022 acquisition](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)).

La transition est techniquement pertinente : l’ancien produit pouvait fonctionner avec une base de données externe ; EPG en reprend les données dans une base intégrée et déplace la maintenance du système d’exploitation et de la base de données à l’intérieur de la frontière de l’appliance. Les runbooks, la supervision et la reprise ne doivent donc pas seulement remplacer les termes de produit, mais réattribuer les surfaces de contrôle ainsi que les données et configurations stockées.

## Sources

- [Kiteworks – FAQ for totemomail Customers Updating to EPG](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)
- [Kiteworks – Top 6 Reasons to Add Email Protection Gateway](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)
- [Kiteworks – Enterprise 8.0 Licensing and Packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)
- [Microsoft Marketplace – totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [NIST CMVP – Certificate 3500, Totemo Cryptographic Module](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)
- [NIST – Totemo Cryptographic Module Security Policy 2019](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)
- [Microsoft – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322)
- [Kiteworks – Top 5 Reasons totemo and Kiteworks Are Better Together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [L-Bank – Communication e-mail sécurisée avec Totemomail](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)
- [Kiteworks – Security and Integration Overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513)
- [IETF RFC 5424 – Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [IETF RFC 5425 – TLS Transport Mapping for Syslog](https://datatracker.ietf.org/doc/html/rfc5425)
- [NVD – CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563)
- [NVD – CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918)
- [NVD – CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Entrust Partner Directory – Totemo](https://www.entrust.com/partners/directory/totemo)
- [NIST – Totemo Cryptographic Module Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf)
- [Kiteworks – The Deal: On the Hunt](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)
