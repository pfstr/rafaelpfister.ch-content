---
title: "Envoi d’e-mails via un relais : vérifier TLS et l’authentification"
navTitle: "Relais : vérifier TLS"
description: "Un guide d’une page pour les responsables d’applications dont l’application envoie des e-mails via un relais : quels sont les trois paramètres importants dans l’application (port, mode TLS, authentification), comment s’appellent les options dans les environnements courants et comment un seul e-mail de test permet de prouver, grâce à l’en-tête Received, que la connexion est réellement chiffrée et authentifiée."
date: "2026-08-28"
kategorie: "SMTP et flux de messagerie"
timeToRead: "5 min de lecture"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "envoi-d-e-mails-via-un-relais-verifier-tls-et-l-authentification"
translationId: "article-734e79c4a87105e3"
translationOf: mail-relay-tls-authentisierung-pruefen
url: https://rafaelpfister.ch/fr/blog/envoi-d-e-mails-via-un-relais-verifier-tls-et-l-authentification
translationSourceHash: 51d48e038c5eb870c77828f954ce1ad1d27bc4758889cb492c872eeaede04d9e
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:29:29.466Z
translationReview: automatic
---

# Envoi d’e-mails via un relais : vérifier TLS et l’authentification

De nombreuses applications n’envoient pas elles-mêmes les e-mails sur Internet, mais les remettent à un relais interne : l’ERP y remet ses confirmations de commande, la supervision ses alertes, le système de tickets ses notifications. L’équipe messagerie exploite le relais ; côté application, le responsable d’application est chargé du sujet. Lors d’un audit ou d’une analyse des besoins de protection, la question lui revient donc : l’application se connecte-t-elle au relais de manière chiffrée et s’authentifie-t-elle correctement ?

La réponse se trouve à deux endroits qui ne nécessitent ni outil de messagerie ni accès au relais : dans la configuration SMTP de sa propre application et dans l’en-tête d’un unique e-mail de test. Ce que le relais propose lui-même et la manière dont il chiffre les e-mails jusqu’au destinataire relèvent de la responsabilité de l’équipe messagerie ; côté application, il suffit de démontrer sa propre portion du trajet.

## Où se trouvent les paramètres

La configuration SMTP se trouve, selon l’application, à l’un de trois emplacements : dans l’interface d’administration (généralement sous « E-mail », « Notifications », « SMTP » ou « Serveur sortant »), dans un fichier de configuration ou dans les variables d’environnement du déploiement (typiquement `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` et leurs variantes). Les mêmes informations sont toujours à rechercher : nom du serveur, port, option de chiffrement et identifiants.

## Les trois paramètres qui comptent

**Premièrement, le port et le mode TLS.** Les deux doivent correspondre, car les valeurs proposées recouvrent deux mécanismes différents : avec STARTTLS, la connexion débute en clair puis passe à TLS ; avec TLS implicite (souvent appelé « SSL/TLS » ou « SSL » dans les interfaces), elle est chiffrée dès le départ.

| Port | Paramètre TLS dans l’application | Évaluation |
|---|---|---|
| 587 | STARTTLS | État attendu pour la remise par des applications |
| 465 | SSL/TLS (implicite) | également correct |
| 25 | aucun ou STARTTLS | courant pour les relais avec autorisation par IP ; activer néanmoins le paramètre TLS si le relais propose STARTTLS |
| quelconque | « Aucun » / « None » | Constat : l’envoi se fait en clair |
| quelconque | « TLS si disponible » / opportuniste | Constat : en cas de problème, retour silencieux au clair ; passer à TLS obligatoire |

Une mauvaise combinaison (par exemple « SSL/TLS » sur le port 587) entraîne des interruptions de connexion, et non du clair non détecté. Les paramètres risqués sont les deux dernières lignes du tableau, car l’application y envoie des e-mails non chiffrés sans message d’erreur.

**Deuxièmement, la vérification du certificat.** De nombreuses applications proposent une option telle que « Ne pas vérifier le certificat », « Allow insecure » ou `verify=false`, souvent activée lors des projets d’introduction parce que le relais utilise un certificat interne. La connexion reste certes chiffrée, mais l’application accepte alors n’importe quel interlocuteur. Si l’option est activée, elle doit figurer comme constat dans le rapport ; la solution propre consiste à faire confiance à l’autorité de certification interne plutôt qu’à désactiver la vérification.

**Troisièmement, l’authentification.** Les relais connaissent deux modèles : SMTP AUTH avec nom d’utilisateur et mot de passe, ou une autorisation par IP sans compte. La variante applicable figure dans l’autorisation du relais fournie par l’équipe messagerie. Pour SMTP AUTH, trois points doivent figurer sur la liste de contrôle : l’authentification utilise un compte de service dédié à l’application (et non un compte personnel qui sera désactivé au prochain départ), le mot de passe est stocké comme secret plutôt qu’en clair dans un fichier de configuration, et l’option TLS est active, car les mécanismes courants PLAIN et LOGIN transmettent sinon les identifiants en clair.

## Noms des paramètres dans les environnements courants

| Environnement | Chiffrement | Authentification |
|---|---|---|
| Interfaces d’administration (ERP, supervision, appliances) | Liste déroulante « Chiffrement » : None / STARTTLS / SSL-TLS | Champs nom d’utilisateur/mot de passe ; vides = pas d’authentification |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` plus `mail.smtp.starttls.required=true`; pour le port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (active STARTTLS) ; MailKit : `SecureSocketOptions.StartTls` | `Credentials` ou `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` pour 587, `'ssl'` pour 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` après l’établissement de la connexion ou `SMTP_SSL` pour 465 | `login()` |
| Node.js (Nodemailer) | Port 465 : `secure:true`; port 587 : `secure:false` plus `requireTLS:true` | `auth: {user, pass}` |

Deux points de ce tableau constituent, d’expérience, les constats les plus fréquents : en Java, `starttls.enable` seul n’active que TLS opportuniste ; seul `starttls.required` empêche le retour au clair. Dans Nodemailer, `secure:false` ne signifie pas « non chiffré », mais « pas de TLS implicite » ; sans `requireTLS:true`, STARTTLS reste toutefois lui aussi opportuniste.

## Contre-vérification : un e-mail de test et son en-tête Received

La configuration indique l’état attendu, mais ne prouve pas ce qui se passe réellement sur le réseau. La preuve figure dans l’en-tête Received que le relais ajoute à la réception de chaque e-mail. Un e-mail de test envoyé par l’application vers sa propre boîte aux lettres suffit ; affichez-y l’en-tête du message (Outlook : Fichier, Propriétés, En-têtes Internet ; Gmail : Afficher l’original) et lisez la ligne Received la plus basse, car les en-têtes croissent de bas en haut :

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Le mot-clé après `with` résume le résultat du contrôle. Les désignations sont normalisées (registre IANA « Mail Transmission Types ») :

| Désignation | Signification | Évaluation |
|---|---|---|
| `SMTP` / `ESMTP` | non chiffré, sans authentification | Mesure requise si TLS est exigé |
| `ESMTPS` | TLS, sans authentification | correct avec une autorisation par IP |
| `ESMTPA` | authentifié, mais sans TLS | critique : les identifiants ont transité en clair |
| `ESMTPSA` | TLS et authentifié | état attendu avec SMTP AUTH |

Postfix et Exchange ajoutent entre parenthèses la version TLS et le chiffrement, ce qui permet également de détecter des versions de protocole obsolètes. Pour analyser des en-têtes plus longs comportant plusieurs étapes, l’[analyseur d’en-têtes e-mail](https://rafaelpfister.ch/tools/header-analyzer) de ce site vous évite le travail manuel ; il fonctionne entièrement localement dans le navigateur, l’en-tête ne quitte pas votre ordinateur.

Si l’en-tête reste ambigu ou qu’un équilibreur de charge en amont modifie le marquage de la connexion, c’est le moment de solliciter l’équipe messagerie : le journal du relais consigne pour chaque remise si TLS a été négocié et avec quel compte l’application s’est authentifiée.

## Liste de contrôle succincte pour le rapport d’audit

1. Configuration SMTP de l’application trouvée (interface, fichier de configuration ou variables d’environnement) et documentée.
2. Le port et le mode TLS correspondent (587/STARTTLS ou 465/SSL-TLS) ; aucun paramètre « Aucun » ou « TLS si disponible ».
3. Vérification du certificat active ; un « Ne pas vérifier le certificat » activé est enregistré comme constat.
4. Modèle d’authentification clarifié : SMTP AUTH avec compte de service et stockage du secret, ou autorisation par IP conformément à l’autorisation du relais.
5. L’en-tête Received de l’e-mail de test affiche `ESMTPSA` (avec compte) ou `ESMTPS` (avec autorisation par IP) ; `ESMTPA` et `ESMTP` constituent des constats.
6. Si le chiffrement jusqu’au destinataire est requis : l’adresser comme exigence à l’équipe messagerie, car le trajet à partir du relais se situe hors de l’application.

## Sources

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): définit STARTTLS et le basculement de la connexion en clair vers TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): définit SMTP AUTH et les mécanismes tels que PLAIN et LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): recommande TLS implicite sur le port 465 pour la remise par les clients.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): registre des désignations `with` dans l’en-tête Received (ESMTPS, ESMTPA, ESMTPSA).

5.  [Jakarta Mail: Package com.sun.mail.smtp](https://jakarta.ee/specifications/mail/2.1/apidocs/jakarta.mail/com/sun/mail/smtp/package-summary): documente les propriétés mail.smtp.starttls.enable, starttls.required et ssl.enable.
