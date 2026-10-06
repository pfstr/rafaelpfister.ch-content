---
title: "Connexion à HIN Mail : webmail, token Outlook, mot de passe oublié et HIN Mail Global"
navTitle: "Connexion à HIN Mail"
description: "Où et comment vous connecter à HIN Mail : webmail sur webmail.hin.ch, token de messagerie pour Outlook et smartphone, connexion sans HIN Client par code SMS ou application Authenticator, réinitialisation du mot de passe et ouverture de HIN Mail Global en tant que destinataire sans compte HIN."
date: "2026-10-05"
kategorie: "Passerelle HIN"
timeToRead: "7 min de lecture"
themen:
  - hin-gateway
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - hin-plattformerneuerung-2026
  - hin-mailgateway-backup-disaster-recovery
slug: "connexion-a-hin-mail-webmail-token-outlook-mot-de-passe-oublie-et-hin-mail-global"
translationId: "article-6f466c817c7c7c66"
translationOf: hin-mail-login
url: https://rafaelpfister.ch/fr/blog/connexion-a-hin-mail-webmail-token-outlook-mot-de-passe-oublie-et-hin-mail-global
translationSourceHash: 8f863faa139fd22182bf59e849447e241e339e4fcc4e6a073e6c1a89344f8a06
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:39:12.105Z
translationReview: automatic
---

# Connexion à HIN Mail : webmail, token Outlook, mot de passe oublié et HIN Mail Global

La recherche « Connexion à HIN Mail » recouvre généralement trois situations différentes : vous êtes membre HIN et souhaitez lire vos e-mails dans le navigateur, vous souhaitez configurer HIN Mail dans Outlook ou sur votre smartphone, ou vous avez reçu, en tant que patiente, patient ou organisme externe, un e-mail HIN chiffré sans disposer de compte HIN. La connexion se déroule différemment dans chacun de ces trois cas.

| Situation | Adresse | Nom d’utilisateur | Mot de passe |
|---|---|---|---|
| Webmail dans le navigateur | `webmail.hin.ch` | ID HIN | Mot de passe HIN, sécurisé par HIN Client, code SMS ou application Authenticator |
| Outlook, Thunderbird, Apple Mail, smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | ID HIN | Token de messagerie provenant de `apps.hin.ch` |
| Administration, token, mot de passe d’initialisation | `apps.hin.ch` | ID HIN | Mot de passe HIN plus second facteur |
| Destinataire sans compte HIN (HIN Mail Global) | Pièce jointe `secure-email.html` ou lien dans l’e-mail | Numéro de mobile | Code SMS |

## Webmail HIN : connexion dans le navigateur

Le webmail est accessible à l’adresse `https://webmail.hin.ch`. Connectez-vous avec votre ID HIN (le nom de connexion HIN, par ex. `hmuster`) et votre mot de passe HIN. Le second facteur est fourni par l’une des méthodes suivantes :

- **HIN Client :** Si le client est lancé et connecté sur le poste de travail, le webmail s’ouvre sans autre demande. Avec HIN Client 4.0, la connexion s’effectue directement dans le navigateur : vous saisissez l’adresse protégée par HIN et êtes redirigé vers la connexion.
- **Code SMS (mTAN) :** Après le mot de passe, vous recevez un code sur le numéro de mobile enregistré. Cette méthode convient aux appareils sans HIN Client, par exemple un ordinateur portable privé.
- **Application HIN Authenticator :** L’application (App Store et Google Play, « HIN Authenticator ») confirme la connexion sur le smartphone.

Le code SMS et l’application Authenticator doivent être activés au préalable. Si aucun des deux n’est disponible et que HIN Client n’est pas accessible, il n’est pas possible de se connecter au webmail. Configurez donc au moins une alternative tant que le client fonctionne encore.

Le webmail permet de rédiger et gérer des e-mails, d’entretenir des contacts et des carnets d’adresses, de rechercher dans l’annuaire des participants HIN et de créer plusieurs signatures. Sous le nom d’utilisateur, une barre indique l’espace libre de la boîte aux lettres ; si elle est rouge, vous devez supprimer des e-mails ou augmenter l’espace de stockage.

## HIN Mail dans Outlook et sur smartphone : connexion avec un token de messagerie

Les clients de messagerie ne se connectent pas avec le mot de passe HIN, mais avec un **token de messagerie**. C’est la cause la plus fréquente des demandes répétées de mot de passe dans Outlook : le mot de passe HIN est refusé lors de la connexion IMAP.

Voici comment générer le token :

1. Ouvrez `https://apps.hin.ch` et connectez-vous avec l’ID HIN (HIN Client, code SMS ou application Authenticator).
2. Sélectionnez la section « HIN Mail ».
3. Cliquez sur « Ajouter un appareil » et choisissez « Mail Clients » comme type d’appareil.
4. Copiez le token affiché et saisissez-le comme mot de passe dans le client de messagerie.

Les paramètres de serveur :

| Paramètre | Courrier entrant (IMAP) | Courrier sortant (SMTP) |
|---|---|---|
| Serveur | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Chiffrement | SSL/TLS | STARTTLS |
| Nom d’utilisateur | ID HIN | ID HIN |
| Mot de passe | Token de messagerie | Token de messagerie |
| Authentification | normale | Activer « Le serveur sortant requiert une authentification » |

Le nom d’utilisateur est l’ID HIN, et non l’adresse e-mail. Lors de la configuration automatique, Outlook propose l’adresse e-mail ; la configuration doit donc être effectuée manuellement via « Options avancées » ou « Configuration manuelle ».

Le token est lié à votre identité HIN et peut être révoqué et régénéré à tout moment dans `apps.hin.ch`. Il est recommandé d’utiliser un token distinct pour chaque appareil : si un smartphone est perdu, vous ne bloquez que cet appareil, tandis qu’Outlook sur le PC du cabinet continue de fonctionner. Une fois le token configuré, Outlook n’a plus besoin de HIN Client pour relever les e-mails.

## Première connexion sans HIN Client : activer l’identité

Les nouvelles identités HIN peuvent être activées sans HIN Client. Vous avez besoin du nom d’utilisateur et du mot de passe d’initialisation figurant dans les documents HIN.

1. Ouvrez le lien d’activation dans le centre de services (`servicecenter.hin.ch`).
2. Saisissez le nom d’utilisateur et le mot de passe d’initialisation.
3. Définissez votre propre mot de passe : au moins 10 caractères, avec des chiffres, des majuscules et minuscules ainsi que des caractères spéciaux.
4. Configurez immédiatement un second facteur (code SMS ou application Authenticator).
5. Connectez-vous ensuite à `apps.hin.ch`.

Vous disposez de 10 minutes pour l’activation. Passé ce délai, le mot de passe d’initialisation est réinitialisé pour des raisons de sécurité et vous devez en demander un nouveau.

## Mot de passe HIN oublié

La réinitialisation dépend de la configuration ou non d’une méthode de connexion alternative.

**Avec code SMS ou application Authenticator :**

1. Connectez-vous à `apps.hin.ch` avec la méthode alternative.
2. Sélectionnez l’onglet « HIN Client » et générez un nouveau mot de passe d’initialisation.
3. Dans HIN Client, sélectionnez « Inscription », puis saisissez le nom de connexion et le nouveau mot de passe d’initialisation.
4. Définissez un nouveau mot de passe et confirmez « Enregistrer une nouvelle identité HIN ».

**Sans méthode alternative et sans appareil connecté**, seul le support téléphonique HIN reste possible (0848 830 740, du lundi au vendredi de 08:00 à 18:00). Notez le nouveau mot de passe d’initialisation avant de fermer la fenêtre.

## HIN Mail Global : ouvrir un e-mail chiffré sans compte HIN

Les cabinets médicaux, hôpitaux et laboratoires envoient des messages aux patientes, patients ou organismes sans affiliation HIN via HIN Mail Global. Vous recevez l’e-mail sous forme d’e-mail ordinaire, mais son contenu se trouve chiffré dans la pièce jointe `secure-email.html` ou derrière le lien « Lire le message chiffré ». Il n’y a pas de connexion HIN à proprement parler ; l’authentification s’effectue avec votre numéro de mobile.

**La première fois :**

1. Téléchargez et ouvrez la pièce jointe `secure-email.html` (ou cliquez sur le lien dans l’e-mail). Avec des services de webmail tels que Gmail, Bluewin ou GMX, enregistrez-la d’abord, puis ouvrez-la.
2. Choisissez la langue et saisissez votre propre numéro de mobile.
3. Saisissez le code reçu par SMS. Le message s’affiche.

**Pour les messages suivants**, il suffit d’ouvrir la pièce jointe ; le code SMS est automatiquement envoyé au numéro enregistré.

À noter :

- Le message est accessible via le lien pendant **30 jours**, après quoi l’accès expire. Enregistrez ou imprimez les contenus importants durant ce délai.
- Seul le destinataire initial peut ouvrir les messages transférés.
- Vous pouvez répondre de manière chiffrée.
- Un appareil peut être marqué comme fiable ; la demande par SMS est alors supprimée pendant un an.
- Si aucun code n’arrive dans les 15 minutes, vous pouvez en demander un nouveau.
- Les deux versions actuelles de Chrome, Safari, Edge et Firefox sont prises en charge.

En cas de problème avec HIN Mail Global, le support HIN Mail peut vous aider au +41 58 670 48 69. Seul l’expéditeur peut répondre aux questions sur le contenu du message.

## Erreurs fréquentes lors de la connexion à HIN Mail

| Symptôme | Cause | Solution |
|---|---|---|
| Outlook demande constamment le mot de passe | Mot de passe HIN saisi au lieu du token de messagerie | Générez un token dans `apps.hin.ch` et saisissez-le comme mot de passe |
| La connexion à Outlook échoue malgré le token | Adresse e-mail utilisée comme nom d’utilisateur | Utilisez l’ID HIN comme nom d’utilisateur |
| L’envoi échoue, la réception fonctionne | Authentification SMTP non activée, port ou chiffrement incorrect | Utilisez le port `587` avec STARTTLS et activez l’authentification |
| Le webmail demande un code qui n’arrive jamais | Numéro de mobile obsolète ou code SMS non activé | Utilisez l’application Authenticator ou connectez-vous via HIN Client et mettez le numéro à jour |
| Le mot de passe d’initialisation est refusé | Délai de 10 minutes dépassé | Demandez un nouveau mot de passe d’initialisation |
| Le token fonctionnait puis s’est soudainement arrêté | Token révoqué dans `apps.hin.ch` ou appareil supprimé | Générez un nouveau token pour cet appareil |
| HIN Mail Global : le lien ne s’ouvre plus | Délai de 30 jours expiré | Demandez à l’expéditeur de renvoyer le message |

## HIN Client 4.0 : ce qui change pour la connexion

HIN déploie automatiquement HIN Client 4.0 jusqu’au 26 octobre 2026 ; si vous ne recevez pas la mise à jour automatique, téléchargez-le à l’adresse `download.hin.ch`. Le mot de passe existant reste valable et doit être confirmé une fois lors de la première connexion avec la nouvelle version. La connexion aux services protégés par HIN, tels que le webmail, s’effectue ensuite directement dans le navigateur.

La version 4.0 nécessite Windows 11 avec TPM ou macOS 14 avec Security Chip. Les appareils ne remplissant pas ces exigences ne peuvent plus utiliser le client. L’accès au webmail reste possible sur ces appareils par code SMS ou application Authenticator, et la relève des e-mails dans Outlook via le token de messagerie. Les détails sur les délais du renouvellement de la plateforme figurent dans l’article [Renouvellement de la plateforme HIN 2026](/blog/hin-plattformerneuerung-2026).

## Cabinets et hôpitaux avec leur propre serveur de messagerie

Les organisations disposant de leur propre passerelle HIN Mail ne se connectent pas à HIN Mail. Les boîtes aux lettres se trouvent sur leur propre Exchange ou dans Exchange Online, et la passerelle (une appliance SEPPmail) chiffre les e-mails à destination et en provenance de la communauté HIN. Les problèmes de connexion concernent alors leur propre système de messagerie, et non la plateforme HIN. Cette passerelle est remplacée par la nouvelle passerelle HIN (« Stargate »).

<aside class="offer-box">
  <span class="offer-box__tag">Vérification gratuite</span>
  <p><strong>Exploitez-vous votre propre passerelle HIN Mail ?</strong> J’examine votre environnement et vous indique ce qui doit être fait avant la migration vers « Stargate ».</p>
  <a class="offer-box__cta" href="/stargate">S’inscrire maintenant</a>
</aside>

## Sources

1.  [Support HIN : HIN Mail et Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): aperçu avec l’adresse du webmail et des instructions pour Outlook, Apple Mail, Thunderbird, iOS et Android.

2.  [Support HIN : comment fonctionne le webmail ?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): fonctions du webmail, affichage de l’espace de stockage.

3.  [Support HIN : instructions du service de token de messagerie pour Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): génération de token dans apps.hin.ch, serveurs IMAP et SMTP, ports et chiffrement.

4.  [Support HIN : qu’est-ce qu’un token ?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): tokens de messagerie et tokens d’accès, lien avec l’identité HIN.

5.  [Support HIN : activer l’identité HIN (sans HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): activation dans le centre de services, délai de 10 minutes, règles de mot de passe, second facteur.

6.  [Support HIN : mot de passe oublié](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): réinitialisation via le mot de passe d’initialisation, support téléphonique sans méthode alternative.

7.  [Support HIN : HIN Mail aux non-membres](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): accessibilité pendant 30 jours, enregistrement SMS, appareils fiables, navigateurs pris en charge, téléphone du support.

8.  [HIN : HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): envoi d’e-mails chiffrés à des destinataires sans affiliation HIN.

9.  [Blog HIN : le nouveau HIN Client est arrivé](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): connexion dans le navigateur, exigences système avec TPM ou Security Chip, déploiement jusqu’au 26 octobre 2026.

10.  [HIN Authenticator dans l’App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): application pour le second facteur.
