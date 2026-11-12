---
title: "RustDesk 1.5.3 : connexion avec approbation, cibles de relais vérifiées et IPC Windows renforcée"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 permet des connexions sans mot de passe avec approbation par le pair, n’accepte les demandes de relais que pour les adresses proposées et authentifie les pipes IPC sous Windows. Les stratégies serveur ne peuvent plus modifier les paramètres sensibles. Une mise à jour de tous les clients est recommandée."
date: "2026-11-11"
kategorie: "Télémaintenance et support"
timeToRead: "5 min de lecture"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-connexion-avec-approbation-cibles-de-relais-verifiees-et-ipc-windows-renforcee"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/fr/blog/rustdesk-1-5-3-connexion-avec-approbation-cibles-de-relais-verifiees-et-ipc-windows-renforcee
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:33:37.918Z
translationReview: automatic
---

# RustDesk 1.5.3 : connexion avec approbation, cibles de relais vérifiées et IPC Windows renforcée

RustDesk a publié la version 1.5.3 du client le 9 novembre 2026, huit jours après la 1.5.1. Aucune version 1.5.2 n’est répertoriée sur la page des releases. Cette version apporte une nouvelle fonction (établissement de connexion avec approbation par le pair), une modification de la logique de relais et 22 corrections de bugs. Plusieurs concernent des processus liés à la sécurité : vérification des adresses de relais et des UUID de relais, authentification des Named Pipes sous Windows, clés de configuration bloquées dans les stratégies serveur et, à nouveau, le transfert de l’installateur sous Windows. Le fabricant ne mentionne ni CVE ni Security Advisory à ce sujet (état au 11 novembre 2026). Cette version s’adresse à tous ceux qui exploitent des clients RustDesk, en particulier sous Windows et avec leur propre serveur. L’état précédent, avec mises à jour signées et Trusted Devices V2, est décrit dans l’article [RustDesk 1.5.1 : mises à jour signées, Trusted Devices V2 et durcissement du transfert de fichiers](/blog/rustdesk-1-5-1).

## Vue d’ensemble des versions

| Version | Date | Type | Points clés |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Release client | Request approval, relais uniquement vers les adresses proposées, authentification IPC sous Windows, clés bloquées dans les stratégies, correctifs d’installateur (F-Secure, Junction), correctifs pour macOS, DRM, Wayland, Android et iOS |

Le serveur open source (`rustdesk-server`) n’a pas reçu de nouvelle version ; la dernière reste la 1.1.16 du 20 juillet 2026.

## Nouvelle fonction : Request approval

Avec la PR #16784, il est possible d’établir une connexion sans mot de passe RustDesk. Le pair doit l’accepter dans une boîte de dialogue. Selon la PR, dans ce mode, le client n’envoie aucun mot de passe, quelle qu’en soit la source, y compris le stockage local ou le carnet d’adresses. Un mot de passe explicitement transmis est ignoré. L’option apparaît dans le sous-menu « Request approval » de la liste déroulante des ID et des menus des cartes de pairs, avec les types de connexion pris en charge par la plateforme et le pair (connexion, transfert de fichiers et autres).

Pour les scripts et raccourcis, la PR mentionne le paramètre de ligne de commande `--request-approval` et le paramètre URI `request-approval=true`. Le mode est conservé lors de la reconnexion, du déplacement d’une session dans une autre fenêtre et lors d’autres connexions depuis la barre d’outils distante. Les connexions normales conservent leur comportement habituel en matière de mot de passe.

Selon la PR, les limites suivantes s’appliquent :

- Les méthodes d’authentification disponibles ne sont pas négociées avec le pair. Si seule l’authentification par mot de passe y est autorisée ou si des restrictions s’appliquent à l’écran de connexion, elles continuent de s’appliquer.
- Si une fenêtre existe déjà pour le même pair et le même type de connexion, RustDesk l’active sans modifier le type d’authentification. Pour une nouvelle connexion avec approbation, fermez d’abord la session existante.
- Avec le multiplexage de tunnel, un tunnel déjà approuvé peut être réutilisé sans nouvelle demande.

La PR ne fournit aucune indication sur la compatibilité avec des versions plus anciennes du pair.

## Relais uniquement vers les adresses proposées

La PR #16858 comble une lacune dans la négociation de relais. Jusqu’à présent, le côté contrôlé reprenait sans vérification l’adresse du relais et l’UUID d’un message `RequestRelay` transmis et se connectait à l’adresse indiquée. Désormais, le côté contrôlé mémorise les adresses de relais qui lui ont été proposées ainsi que le relais qu’il choisirait lui-même (32 entrées au maximum, valables dix minutes à compter du dernier message). Les demandes utilisant un autre relais sont rejetées avant la résolution de noms. Les deux côtés rejettent également les UUID vides ou de plus de 128 octets, ou contenant des caractères autres que des lettres ASCII, des chiffres, `-` et `_`. Selon la PR, cela correspond à la vérification déjà effectuée par hbbs.

Une demande rejetée ne reçoit aucune réponse ; la tentative de relais du contrôleur expire donc par timeout plutôt que par un message d’erreur. Un relais retiré de votre configuration n’est plus accepté au plus tard dix minutes après son dernier message.

Pour les opérateurs de leur propre serveur, une restriction est pertinente : si hbbs s’exécute avec l’option marquée comme obsolète `--mask`, le repli vers un relais après une tentative WebRTC échouée est rejeté lorsque hbbs a réécrit l’adresse de relais pour le LAN. Ce point était encore ouvert lors de la fusion. Vérifiez si votre instance hbbs utilise `--mask` et testez les connexions de relais après la mise à jour.

## Autres corrections liées à la sécurité

- **IPC Windows (PR #16755) :** selon la PR, d’autres utilisateurs pouvaient jusqu’à présent créer d’autres instances serveur des Named Pipes pour l’IPC principal et de service, et ainsi intercepter des connexions à RustDesk, lire les données IPC ou falsifier des réponses de configuration. Les clients ouvrent désormais les pipes sans ce droit et authentifient le serveur avant l’échange de données. Les pipes de service doivent appartenir à SYSTEM. Selon la PR, un déni de service reste possible si un attaquant réserve d’abord le nom du pipe, et les versions plus anciennes avec des règles d’accès plus ouvertes ne peuvent pas établir de connexion avant la mise à jour.
- **Stratégies (PR #16696) :** les stratégies distribuées par le serveur via heartbeat ne peuvent plus modifier les paramètres sensibles. Sont notamment bloqués `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` et `websocket-port`. Restent configurables, par exemple, `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` et `allow-auto-update`. Les clés rejetées sont consignées sans leur valeur. L’auteur qualifie explicitement cette modification de durcissement limité : les réponses de stratégie ne sont toujours pas authentifiées.
- **Installateur Windows (PR #16889, #16908, #16871) :** le transfert via Named Pipe introduit avec la 1.5.1 a de nouveau été supprimé, car il empêchait, selon la PR, l’installation sur des systèmes avec F-Secure DeepGuard et « Stop service » (erreur `0x800700E8`). La protection des fichiers préparés repose désormais sur les DACL restrictives de la 1.5.1 et sur des répertoires de staging dédiés, dont les handles restent ouverts pendant le transfert. Si `C:\Windows\Temp` est redirigé via une Junction, l’installateur utilise à la place le dossier ProgramData du système, plutôt que d’interrompre avec une erreur.
- **macOS (PR #16729, #16879) :** le Privacy Mode est désactivé à la fin de la session avant le verrouillage, et le démon de service démarre directement au lieu de passer par `sh`.
- **Liste blanche IP (PR #16692) :** les espaces autour des entrées de la liste blanche IP sont ignorés.

Si l’installation échoue sous Windows avec F-Secure, testez la 1.5.3 sur un appareil concerné avant un déploiement généralisé. Selon la PR #16889, la confirmation du correctif F-Secure par le rapporteur était encore en attente lors de la fusion. Si vous distribuez jusqu’à présent des valeurs de configuration telles que des serveurs API, ID ou de relais via des stratégies, vous devrez désormais les définir par un autre moyen.

## Corrections de bugs par plateforme

- **Windows :** la désinstallation se poursuit après des erreurs de synchronisation de configuration, l’IPC de service est réessayée avant la fermeture de l’interface graphique, le marqueur de propriétaire du presse-papiers est écrit en premier.
- **Linux :** l’écran de connexion sous greetd avec Wayland est fourni via DRM, les contenus d’écran inchangés ne sont plus encodés sous DRM, les erreurs de capture Wayland non attribuées apparaissent avec leur message réel au lieu de « x11 expected ».
- **Android :** plus de plantage avec des contenus de presse-papiers dépassant 1 Mo, les frames sont copiées avant la libération de l’image, l’hébreu et l’indonésien sont correctement associés à partir des anciens codes locale.
- **iOS :** l’écran de démarrage suit le Dark Mode du système.
- **Autres :** le port forwarding maintient le handle KCP ouvert et envoie un motif de fermeture, les barres latérales de la fenêtre d’acceptation restent sur le moniteur actuel sous Windows, les périphériques d’entrée audio ne sont recherchés que parmi les périphériques d’entrée, la taille minimale de la fenêtre principale Sciter n’est plus mise à l’échelle deux fois.

## Problèmes connus

- **Protection contre le rollback pour les mises à jour signées (depuis 1.5.1) :** toujours ouverte. Les notes de version de la 1.5.3 ne mentionnent aucune modification à ce sujet.
- **Variante portable sous Windows (depuis 1.5.1) :** selon la PR #15914, le durcissement avait été déplacé vers une branche distincte. Les notes de version de la 1.5.3 ne mentionnent aucune modification à ce sujet.
- **Trusted Devices V2 (depuis 1.5.1) :** les limites connues (révocation asynchrone, expiration selon l’horloge système de l’hôte) ne sont pas abordées dans les notes de version de la 1.5.3.
- **Repli de relais avec `--mask` (nouveau) :** toujours ouvert selon la PR #16858.
- **Nom du pipe réservé à l’avance (nouveau) :** un déni de service reste possible selon la PR #16755.
- **Demande UAC interrompue (nouveau) :** selon la PR #16889, les fichiers préparés restent dans le répertoire protégé et peuvent être supprimés par un administrateur.
- **Réponses de stratégie (nouveau) :** toujours non authentifiées. Dans les déploiements utilisant HTTP au lieu de HTTPS, les options toujours autorisées peuvent être manipulées selon la PR #16696.

## Procédure recommandée

La version cible pour tous les clients est la 1.5.3. Les versions plus anciennes ne pouvant, selon la PR #16755, établir aucune connexion avec les pipes IPC renforcés avant la mise à jour, mettez conjointement à jour sous Windows tous les composants RustDesk d’un appareil. Avant le déploiement, vérifiez si vous distribuez des adresses de serveur ou des clés via des stratégies, si hbbs s’exécute avec `--mask` et si F-Secure est utilisé sur les systèmes cibles. Le serveur API doit être accessible via HTTPS, comme déjà recommandé pour la 1.5.1. Le serveur open source ne doit pas être mis à jour pour cette version. Les notes de version ne donnent aucune indication sur les périodes de support (état au 11 novembre 2026).

## Sources

1.  [GitHub : RustDesk Release 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3) : notes de version avec la date, la nouvelle fonction, la modification et les corrections de bugs.

2.  [GitHub : RustDesk Releases](https://github.com/rustdesk/rustdesk/releases) : liste des versions sans release 1.5.2.

3.  [GitHub : PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784) : déroulement, paramètres et limites de la connexion avec approbation.

4.  [GitHub : PR #16858, relais uniquement vers les adresses proposées](https://github.com/rustdesk/rustdesk/pull/16858) : liste d’autorisation des adresses de relais, vérification des UUID et restriction avec `--mask`.

5.  [GitHub : PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755) : sécurisation des Named Pipes, compatibilité et risque restant.

6.  [GitHub : PR #16696, clés bloquées dans les stratégies](https://github.com/rustdesk/rustdesk/pull/16696) : options bloquées et toujours autorisées, absence d’authentification des réponses.

7.  [GitHub : PR #16889, suppression du transfert par pipe dans l’installateur](https://github.com/rustdesk/rustdesk/pull/16889) : lien avec F-Secure DeepGuard et comportement en cas de demande UAC interrompue.

8.  [GitHub : PR #16908, protection du répertoire de l’installateur](https://github.com/rustdesk/rustdesk/pull/16908) : répertoires de staging dédiés avec handles ouverts.

9.  [GitHub : PR #16871, staging de l’installateur avec Junction](https://github.com/rustdesk/rustdesk/pull/16871) : recours à ProgramData lorsque Windows Temp est redirigé.

10.  [GitHub : RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories) : aucun advisory publié (état au 11 novembre 2026).

11.  [GitHub : rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases) : dernière version serveur 1.1.16 du 20 juillet 2026.

12.  [RustDesk 1.5.1 : mises à jour signées, Trusted Devices V2 et durcissement du transfert de fichiers](/blog/rustdesk-1-5-1) : état de la version précédente.
