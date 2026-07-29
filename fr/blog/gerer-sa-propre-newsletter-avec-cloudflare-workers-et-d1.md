---
title: "Gérer sa propre newsletter avec Cloudflare Workers et D1"
navTitle: "Newsletter sur Workers"
description: "Le template ouvert fournit l’inscription, le désabonnement, la file d’attente et la base de données dans votre propre compte Cloudflare. Un bouton de déploiement configure Worker, D1 et CI sans serveur local."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "8 min de lecture"
themen:
  - "cloudflare-workers"
slug: "gerer-sa-propre-newsletter-avec-cloudflare-workers-et-d1"
translationOf: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/fr/blog/gerer-sa-propre-newsletter-avec-cloudflare-workers-et-d1"
---

# Gérer sa propre newsletter avec Cloudflare Workers et D1

Avec un service de newsletter hébergé, la liste des destinataires est détenue par le fournisseur, et les coûts augmentent souvent avec le nombre d’abonnés. Un serveur propre offre davantage de contrôle, mais implique un travail continu : mises à jour, surveillance, sauvegardes et exploitation d’un système qui n’envoie peut-être qu’une fois par semaine.

Pour ce cas d’utilisation léger, des points de terminaison HTTP, une petite base de données et une tâche d’envoi planifiée suffisent. Cloudflare Workers et D1 fournissent précisément ces composants. Mon template ouvert les configure dans votre propre compte via un **bouton Deploy to Cloudflare**. Aucune ligne de commande locale ni serveur à maintenir durablement ne sont nécessaires. Le code source sous licence MIT se trouve sur [GitHub](https://github.com/pfstr/newsletter-template).

[![Déployer sur Cloudflare](../images/serverloser-newsletter-cloudflare-workers-d1/deploy-to-cloudflare.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template)

![Le formulaire d’inscription hébergé du template](../images/serverloser-newsletter-cloudflare-workers-d1/newsletter-template-signup.png)

## Ce que peut faire le template

- **Inscription** : une page d’inscription hébergée, un formulaire intégrable sur votre propre site web et un point de terminaison JSON
- **Désabonnement en un clic** : conforme à la RFC 8058, avec un jeton individuel par abonné
- **Mentions obligatoires intégrées** : chaque e-mail reçoit automatiquement un pied de page avec lien de désabonnement et adresse postale ; les dates de consentement et de désabonnement sont enregistrées
- **Envoi** : sur une page protégée, vous pouvez saisir l’objet et le HTML, envoyer un e-mail de test et mettre la campagne en file d’attente ; une tâche d’arrière-plan envoie par lots et répète les tentatives échouées
- **Vos propres données** : les abonnés se trouvent dans une base de données D1 de votre compte et peuvent être exportés à tout moment
- **Optionnel, désactivé par défaut** : double opt-in, protection contre les bots via Turnstile et envoi automatique des nouveaux articles de blog à partir du flux RSS

## Architecture : un Worker, une base de données

L’ensemble du système est un unique Cloudflare Worker avec deux gestionnaires : `fetch` pour HTTP (routé avec Hono) et `scheduled` pour le déclencheur Cron, plus une base de données D1. Il n’y a pas de second service, pas de courtier de file d’attente séparé, pas de backend d’administration dédié ; même la file d’attente d’envoi n’est qu’une table D1.

| Route | Fonction |
| --- | --- |
| `GET /` | Page d’inscription hébergée |
| `GET /embed` | Formulaire transparent à intégrer via iframe |
| `POST /api/subscribe` | Inscription (CORS ouvert pour votre propre site web) |
| `GET /confirm` | Lien de confirmation pour le double opt-in |
| `GET/POST /unsubscribe` | Désabonnement : page de confirmation via GET, exécution via POST (en un clic selon la RFC 8058) |
| `GET /admin` | Page d’envoi (formulaire) |
| `POST /api/send` | Mettre une campagne en file d’attente, protégé par jeton d’administration |

Le modèle de données comprend quatre tables : `subscribers` (e-mail comme clé primaire, nom, statut, jetons de désabonnement et de confirmation, une colonne JSON pour des champs supplémentaires définis par l’utilisateur ainsi que des horodatages pour la confirmation et le désabonnement), `campaigns` avec l’objet, le contenu et les compteurs par envoi, `outbox` comme file d’attente d’envoi (une ligne par destinataire) et `sent_posts` pour la déduplication des envois RSS.

## Déploiement sans ligne de commande

La partie la plus intéressante n’est pas le code, mais le chemin vers un système opérationnel. Le bouton Deploy to Cloudflare lit la configuration Wrangler du dépôt et effectue toute la mise en place : il clone le dépôt dans votre propre compte GitHub, provisionne la base de données D1, exécute les migrations de schéma et configure CI afin que chaque push soit automatiquement déployé. Depuis juillet 2025, le flux de déploiement demande également les variables d’environnement et les secrets directement dans le formulaire : dans le cas de ce template, le mot de passe d’administration (`ADMIN_TOKEN`), le nom et l’adresse de l’expéditeur, le commutateur de double opt-in et la taille des lots d’envoi (`SEND_BATCH`).

Le résultat après un clic et un formulaire : la page d’inscription est en ligne sous `https://<worker-name>.workers.dev` et collecte des abonnés. Aucun terminal n’est ouvert à aucun moment.

## Collecter des abonnés

Pour l’intégration sur votre propre site web, il existe trois options, par ordre croissant de profondeur d’intégration. La plus simple : partager le lien vers la page d’inscription hébergée. La plus pratique pour les constructeurs de sites (WordPress, Webflow, Squarespace, Framer) : une ligne iframe dans n’importe quel bloc d’intégration HTML.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Si vous souhaitez un formulaire dans votre propre design, publiez directement vers le point de terminaison :

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Abonnieren</button>
</form>
```

Le formulaire collecte par défaut l’e-mail et, facultativement, le nom. Vous définissez des champs supplémentaires (entreprise, pays, …) dans un seul fichier (`src/fields.ts`) ; ils apparaissent automatiquement dans les deux formulaires et sont enregistrés comme JSON dans la base de données.

## Envoi : votre propre fournisseur plutôt qu’un prestataire intégré

Pour l’envoi d’e-mails, le template fait un choix délibéré : il est **agnostique vis-à-vis des fournisseurs**. Le fichier `src/email.ts` contient un seul adaptateur `sendEmail()` avec un exemple commenté pour une API HTTP générique. Le service d’envoi que vous y connectez reste votre choix. Aucun fournisseur n’est intégré en dur, aucune inscription à un service précis n’est requise. La collecte d’abonnés fonctionne déjà entièrement sans configuration d’envoi ; l’envoi est activé dès que l’adaptateur est implémenté et que le secret du fournisseur est défini. Si le fournisseur propose en outre un point de terminaison par lots (un appel API, de nombreux e-mails), un adaptateur facultatif `sendEmailBatch()` peut être ajouté dans le même fichier ; un exemple commenté est également fourni.

L’envoi est géré via la page `/admin` : insérez l’objet et le HTML de l’e-mail, envoyez un test à votre propre adresse, puis mettez la campagne en file d’attente pour tous les abonnés. Les balises de fusion `{{unsubscribe_url}}`, `{{email}}` et `{{name}}` sont disponibles dans les e-mails.

L’envoi proprement dit a lieu en arrière-plan, selon le modèle Transactional Outbox : `POST /api/send` écrit la campagne et une ligne par destinataire dans la base de données, puis répond immédiatement. Une tâche Cron exécutée chaque minute délivre ensuite `SEND_BATCH` e-mails par exécution, 40 par défaut : ce choix garantit que chaque exécution reste dans les limites de sous-requêtes du plan Workers Free. Les lignes sont revendiquées de manière atomique, de sorte que des exécutions qui se chevauchent ne peuvent jamais envoyer deux fois ; les livraisons échouées sont réessayées jusqu’à trois fois, et les exécutions interrompues sont reprises après dix minutes. Et toute personne qui se désabonne alors que son e-mail est encore dans la file d’attente ne le reçoit plus : le désabonnement annule également les messages déjà mis en file d’attente.

## Le désabonnement et les preuves font partie du cœur du système

Quiconque envoie une newsletter est soumis aux législations anti-spam et de protection des données : le CAN-SPAM Act américain, le RGPD et ePrivacy dans l’UE, ainsi que la LCD en Suisse. Une part essentielle de ce pour quoi les services de newsletter sont payés est précisément le respect de ces obligations. Le template en prend en charge la partie mécanique :

- **Pied de page obligatoire** : chaque e-mail de campagne reçoit automatiquement un pied de page avec un lien de désabonnement fonctionnel et l’adresse postale de l’expéditeur (`SENDER_ADDRESS`) ; le CAN-SPAM exige une adresse physique dans les e-mails commerciaux. La page d’envoi avertit tant que l’adresse manque.
- **En-têtes List-Unsubscribe selon la RFC 8058** pour chaque envoi : le bouton de désabonnement natif dans Gmail et Outlook, que Gmail et Yahoo exigent des expéditeurs de masse depuis 2024. L’application compose entièrement les en-têtes ; votre propre adaptateur de fournisseur ne fait que les transmettre.
- **Désabonnement résistant aux scanners** : le lien de désabonnement mène à une page de confirmation avec un seul bouton. Les scanners d’e-mails d’entreprise qui consultent à l’avance chaque lien d’un e-mail ne peuvent ainsi désabonner personne par inadvertance ; les clients de messagerie utilisent directement le POST en un clic.
- **Minimisation des données et preuve** : un désabonnement prend effet immédiatement, supprime le nom et les champs supplémentaires et est enregistré avec un horodatage, tout comme l’inscription et la confirmation du double opt-in. Le consentement peut ainsi être prouvé ultérieurement (obligation de responsabilité du RGPD).
- **Lien de confidentialité** : lorsque `PRIVACY_URL` est défini, un lien vers votre propre déclaration de confidentialité apparaît sous le formulaire d’inscription.

Il incombe à l’exploitant de conserver des lignes d’expéditeur et d’objet véridiques, de n’envoyer qu’aux adresses effectivement inscrites et d’assurer l’authentification du domaine (SPF/DKIM/DMARC) auprès du service d’envoi. Rien de tout cela ne constitue un conseil juridique.

## Options : double opt-in, Turnstile, automatisation RSS

Trois fonctions sont intégrées, mais désactivées par défaut afin que le système reste opérationnel sans configuration :

- **Double opt-in** (`DOUBLE_OPT_IN = "true"`) : les nouveaux abonnés sont enregistrés comme `pending` et ne deviennent actifs qu’après un clic sur un lien de confirmation. Pour la Suisse (LPD) et l’UE, cette procédure est le choix le plus rigoureux.
- **Protection contre les bots** avec Cloudflare Turnstile : il suffit de définir la clé de site et la clé secrète comme variables ; le widget apparaît automatiquement sur les deux formulaires, et le Worker vérifie chaque inscription côté serveur. Sans jeton valide, l’inscription est refusée.
- **Envoi RSS automatique** : une tâche Cron vérifie toutes les 15 minutes le flux de votre propre blog (RSS 2.0 ou Atom) et met automatiquement les nouveaux articles en file d’attente d’envoi. Deux protections sont intégrées : lors de la toute première exécution, le flux existant est seulement marqué comme référence (les archives ne sont donc pas envoyées comme un déluge d’e-mails), et chaque identifiant d’article est conservé dans `sent_posts`, afin qu’aucun article ne parte deux fois.

## Limites

Le template est volontairement minimal. L’envoi via file d’attente délivre par défaut environ 40 e-mails par minute avec le plan Free ; une campagne à 1’000 destinataires prend donc environ 25 minutes, ce qui n’a aucune importance pour une newsletter. Avec le plan Workers payant (10’000 sous-requêtes par appel au lieu de 50), `SEND_BATCH` peut être augmenté à plusieurs centaines ; avec un adaptateur par lots (un appel API, jusqu’à environ 1’000 e-mails), le plan Free envoie lui aussi de grandes listes en quelques minutes. La délivrabilité dépend, comme dans tout système, de votre propre domaine expéditeur : SPF, DKIM et DMARC doivent être vérifiés auprès du service d’envoi choisi, faute de quoi la newsletter finira dans les spams. Et le single opt-in par défaut est le démarrage le plus simple, mais pas l’option de conformité la plus prudente ; le commutateur existe pour cela.

Concernant les coûts : Workers et D1 disposent de quotas Free Tier généreux (notamment 100’000 requêtes par jour), qu’un formulaire d’inscription et des envois hebdomadaires à une petite ou moyenne liste n’épuisent pas. Si une limite est atteinte, Cloudflare limite le débit dans le plan Free au lieu d’émettre une facture.

## Essayer

Le code source, y compris le bouton de déploiement, se trouve sur [GitHub](https://github.com/pfstr/newsletter-template) ; vous y trouverez également la documentation complète des variables de configuration.

[![GitHub : pfstr/newsletter-template](../images/serverloser-newsletter-cloudflare-workers-d1/github-newsletter-template.svg)](https://github.com/pfstr/newsletter-template)

## Sources

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template) : code source du template (MIT) avec bouton de déploiement et documentation.

2.  [Boutons Deploy to Cloudflare](https://developers.cloudflare.com/workers/platform/deploy-buttons/) : provisionnement automatique des ressources, clonage du dépôt et CI lors du déploiement.

3.  [Boutons de déploiement : variables d’environnement et secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/) : les secrets et les variables sont demandés dans le formulaire de déploiement depuis juillet 2025.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/) : SQLite sans serveur, utilisé ici pour les abonnés, le journal d’envoi et la déduplication RSS.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) : protection contre les bots sans énigmes CAPTCHA, activable en option dans le template.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058) : signalisation de la fonctionnalité en un clic pour les en-têtes d’e-mails de liste ; base du bouton de désabonnement natif dans Gmail et Outlook.

7.  [Limites de Workers](https://developers.cloudflare.com/workers/platform/limits/) : limites de sous-requêtes par appel (50 dans le plan Free, 10’000 dans le plan payant) ; la taille des lots d’envoi via file d’attente en découle.

8.  [FTC : guide de conformité au CAN-SPAM Act](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business) : obligations pour les e-mails commerciaux, notamment l’adresse postale et un désabonnement fonctionnel.
