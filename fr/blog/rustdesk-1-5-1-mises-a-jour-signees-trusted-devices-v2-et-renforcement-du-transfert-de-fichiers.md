---
title: "RustDesk 1.5.1 : mises à jour signées, Trusted Devices V2 et renforcement du transfert de fichiers"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 vérifie les mises à jour de bureau à l’aide de métadonnées signées, introduit Trusted Devices V2 et renforce le transfert de fichiers, l’installateur Windows et la connexion automatique. Près de 35 corrections s’y ajoutent, principalement pour la capture DRM et Wayland. Une mise à jour des clients est recommandée, le serveur reste inchangé."
date: "2026-11-02"
kategorie: "Télémaintenance et support"
timeToRead: "5 min de lecture"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-mises-a-jour-signees-trusted-devices-v2-et-renforcement-du-transfert-de-fichiers"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/fr/blog/rustdesk-1-5-1-mises-a-jour-signees-trusted-devices-v2-et-renforcement-du-transfert-de-fichiers
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:34:51.448Z
translationReview: required
---

# RustDesk 1.5.1 : mises à jour signées, Trusted Devices V2 et renforcement du transfert de fichiers

RustDesk a publié la version 1.5.1 du client le 1er novembre 2026, un peu plus d’un mois après la version 1.5.0 (30 septembre 2026). Cette version apporte cinq nouvelles fonctions, dont la vérification des métadonnées de mise à jour signées et une nouvelle version de Trusted Devices pour l’authentification à deux facteurs, trois modifications et près de 35 corrections. Plusieurs d’entre elles renforcent des processus importants pour la sécurité : validation des chemins de téléchargement, lancement protégé de l’installateur sous Windows, avertissement en cas de serveurs API non chiffrés et connexion automatique plus restrictive. Le fabricant ne mentionne ni CVE ni avis de sécurité à ce sujet (état au 2 novembre 2026). Cette version s’adresse à toutes les personnes utilisant des clients RustDesk sous Windows, macOS, Linux, Android ou iOS, en particulier avec leur propre serveur. La configuration et l’exploitation de votre propre serveur RustDesk sont décrites dans l’article [RustDesk : configurer l’alternative open source à TeamViewer](/blog/rustdesk-teamviewer-alternative).

## Aperçu des versions

| Version | Date | Type | Points principaux |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Version client | métadonnées de mise à jour signées, Trusted Devices V2, raccourcis clavier, validation des chemins de téléchargement, renforcement de l’installateur sous Windows, avertissement en cas de serveur API HTTP, correctifs pour DRM, Wayland, X11 et QSV |

Le serveur open source (`rustdesk-server`) n’est pas concerné par cette version. Sa dernière version est la 1.1.16 du 20 juillet 2026.

## Métadonnées de mise à jour signées

Avec la PR #15914, le programme de mise à jour de bureau sous Windows et macOS obtient ses informations à partir de métadonnées de version signées. Chaque version contient à cet effet les ressources `rustdesk-update.json` (version, URL, nom de fichier, taille et SHA-256 par artefact) et `rustdesk-update.json.sig` (signature Ed25519). Le client vérifie la signature avant d’évaluer les métadonnées et n’accepte que les URL de téléchargement relevant de la version de `rustdesk/rustdesk`. Si la vérification échoue, la mise à jour est interrompue ; aucun repli sur des données non signées n’est prévu. Les installateurs téléchargés sont conservés dans un répertoire protégé jusqu’à leur lancement et font de nouveau l’objet d’un hachage avant une exécution avec des droits élevés.

Selon la PR, la signature atteste de l’authenticité et de l’intégrité, mais non de l’actualité : aucune protection contre la réinstallation d’anciennes versions correctement signées (rollback) n’est incluse. La variante portable sous Windows continue à extraire et à démarrer depuis un répertoire accessible en écriture par l’utilisateur ; selon la PR, son renforcement a été reporté dans une branche séparée.

## Trusted Devices V2

La PR #16548 remplace l’ancienne relation de confiance basée sur l’identifiant matériel par des jetons émis par l’hôte. Une nouvelle inscription nécessite toujours le mot de passe permanent et le TOTP ; un jeton Trusted Device dispense ensuite uniquement de saisir le TOTP. L’hôte ne stocke que le hachage SHA-256 du jeton sous forme chiffrée, tandis que le contrôleur lie son jeton à la clé publique vérifiée de l’hôte. Aucune relation de confiance n’est proposée pour les connexions IP directes, car l’identité de l’hôte n’y est pas vérifiée.

Pour l’exploitation, les points suivants de la PR sont importants :

- **Validité :** réglable sur 1, 7, 30 ou 90 jours, ou illimitée ; la valeur par défaut est de 90 jours. Les relations de confiance peuvent être renouvelées, supprimées individuellement ou effacées collectivement dans les paramètres.
- **Révocation :** la modification du mot de passe, de la 2FA ou du paramètre Trusted Device révoque les jetons concernés. L’hôte conserve au maximum 128 entrées.
- **Versions mixtes :** les relations de confiance existantes et non expirées restent valides après la mise à jour de l’hôte. Les nouvelles inscriptions nécessitent la version 1.5.1 des deux côtés ; dans les sessions mixtes, RustDesk demande le TOTP et ne propose pas « Trust this device ».
- **Rétrogradation :** selon la PR, elle n’est pas sûre, car les versions antérieures ne peuvent pas lire les relations de confiance importées.
- **Limites connues :** une révocation est d’abord enregistrée localement, puis transférée de manière asynchrone dans la configuration racine ; un redémarrage avant la synchronisation peut restaurer une ancienne relation de confiance. La date d’expiration dépend de l’horloge système de l’hôte.

Planifiez donc la mise à jour de manière à ce que les hôtes et les contrôleurs passent rapidement à la version 1.5.1, et évitez de revenir à la version 1.5.0.

## Autres corrections importantes pour la sécurité

- **Chemins de téléchargement (PR #16601) :** les téléchargements refusent les noms de fichiers et chemins non sûrs qui sortent du répertoire cible sélectionné ou passent par des liens symboliques. Les listes de fichiers distants sont vérifiées avant le téléchargement ; l’écrasement de fichiers existants et la reprise de téléchargements interrompus nécessitent une confirmation.
- **Installateur Windows (PR #16554, #16562) :** les scripts destinés au lancement de l’installateur avec des droits élevés ne se trouvent plus dans `System32`, mais dans un répertoire sous Windows Temp avec une DACL restreinte (Administrators et SYSTEM). Le service est contrôlé via le contrôle natif des services. L’une des raisons était notamment que Bitdefender bloquait le script Batch temporaire (issue #16557).
- **Serveur API non chiffré (PR #16616) :** les paramètres du serveur affichent un avertissement lorsque le serveur API est vide ou utilise une adresse HTTP distante. HTTPS et les adresses locales ne déclenchent aucun avertissement. La connexion n’est pas bloquée.
- **Connexion automatique (PR #16520) :** la connexion automatique ne saisit plus le mot de passe du système d’exploitation que sur l’écran de connexion ou de verrouillage.
- **Mode confidentialité (PR #16507) :** une demande de mode confidentialité refusée ne désactive plus le mode confidentialité d’une autre connexion.

Si vous exploitez votre propre serveur avec une adresse `http://` comme serveur API, le nouvel avertissement s’affichera pour vos utilisateurs. Passez le serveur API à HTTPS avant de déployer la version 1.5.1 afin d’éviter les questions.

## Nouvelles fonctions et modifications

Outre les éléments déjà mentionnés, la version 1.5.1 apporte des raccourcis clavier (PR #14933), un bouton dans le carnet d’adresses pour ouvrir la console Web et, sous Android, le transfert direct de fichiers vers des dossiers SAF. L’encodage des contenus d’images statiques après le défilement a été modifié (affichage plus net), tout comme les budgets de rafraîchissement de MediaCodec. La prise en charge de `ConfigUpdate` a été supprimée.

## Corrections par plateforme

- **Linux, capture DRM :** taille correcte pour les écrans pivotés, pas de réveil des écrans en veille au démarrage, suppression des images en double et fréquence d’images adaptée à l’encodeur.
- **Linux, Wayland :** capture PipeWire sans attente, attribution correcte du flux de portail à la sortie, disposition du clavier lors de la saisie, et une demande de portail ouverte ne bloque plus le serveur.
- **Linux, général :** utilisateur actif correct sous Flatpak, plantage lors de la fermeture de la fenêtre GTK, demande sudo sans faux « Sorry, try again. » et prise en charge de sudo-rs, noms de sorties xrandr, interrogation des moniteurs RandR sous X11.
- **Windows :** lecture de `host=` depuis des chemins de programme contenant des caractères non ASCII, exécutable explicitement nommé pour la fenêtre de confidentialité, blocages de l’encodeur QSV.
- **macOS :** lien vers les paramètres d’enregistrement de l’écran, relâchement du bouton de la souris.
- **Mobile :** plus aucun Mouse-Down retardé après le relâchement du bouton flottant.
- **Réseau :** erreur HTTP API 407 via des proxys HTTPS, serveur d’identifiants IPv6 sans crochets lors de la déduction du serveur API, un message d’erreur au lieu de deux lorsque les deux tentatives de connexion WebRTC échouent.
- **Autres :** les imprimantes supprimées sont conservées, buffer de relecture dans le terminal, navigation au clavier dans la liste de fichiers, chargement des pairs sans répertoire d’application défini, clics sur les cartes de pairs dans le client Web.

## Procédure recommandée

La version cible pour tous les clients est la 1.5.1. Mettez à jour les hôtes et les contrôleurs autant que possible durant la même période afin que les nouvelles inscriptions Trusted Device fonctionnent ; selon la PR #16548, une rétrogradation vers la version 1.5.0 n’est pas sûre. Avant le déploiement, vérifiez que votre propre serveur API est accessible via HTTPS. Le serveur open source ne doit pas être mis à jour pour cette version. Les notes de version ne contiennent aucune information sur les problèmes connus ni sur les délais de support (état au 2 novembre 2026).

## Sources

1.  [GitHub : RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1) : notes de version avec la date, les nouvelles fonctions, les modifications et les corrections.

2.  [GitHub : RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0) : version précédente du 30 septembre 2026.

3.  [GitHub : PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914) : processus de vérification de la signature, protection rollback absente, variante portable.

4.  [GitHub : PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548) : modèle de jetons, compatibilité avec les versions antérieures, rétrogradation et limites connues.

5.  [GitHub : PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601) : vérification des noms de fichiers, chemins et liens symboliques lors des téléchargements.

6.  [GitHub : PR #16554, scripts d’installateur dans des répertoires temporaires protégés](https://github.com/rustdesk/rustdesk/pull/16554) : déplacement depuis System32, DACL et lien avec Bitdefender.

7.  [GitHub : PR #16616, avertissement en cas de serveur API non sûr](https://github.com/rustdesk/rustdesk/pull/16616) : conditions et texte du nouvel avertissement.

8.  [GitHub : RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories) : aucun avis publié (état au 2 novembre 2026).

9.  [GitHub : rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases) : dernière version du serveur, 1.1.16, du 20 juillet 2026.

10.  [RustDesk : configurer l’alternative open source à TeamViewer](/blog/rustdesk-teamviewer-alternative) : configuration du client et de votre propre serveur.
