---
title: "Proofpoint Protection Server 8.22.4 : un correctif peut réinitialiser la configuration du certificat pour le client SMTP"
navTitle: "PPS 8.22.4 certificat"
description: "Si, dans Proofpoint Protection Server, sous System > Certificates > Services, l’association « SMTP Client » est définie sur « -none- », Sendmail effectue les livraisons sans certificat client TLS. Les serveurs distants qui vérifient les certificats les refusent alors, par exemple Exchange Online avec 5.7.51 ou un Smarthost avec « Relay access denied ». Voici comment vérifier et corriger le problème."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min de lecture"
themen:
  - proofpoint
  - smtp-mailflow
produkte:
  - "proofpoint-pps"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-protection-server-8-22-4-un-correctif-peut-reinitialiser-la-configuration-du"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
url: https://rafaelpfister.ch/fr/blog/proofpoint-protection-server-8-22-4-un-correctif-peut-reinitialiser-la-configuration-du
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:26:35.733Z
translationReview: automatic
---

J’ai analysé aujourd’hui une panne du flux de messagerie sortant sur un Proofpoint Protection Server (PPS) 8.22.4. Conclusion rapide : après l’installation automatique d’un correctif, sous **System > Certificates > Services**, l’association **SMTP Client** était définie sur `-none-`. Sendmail a alors établi des connexions sortantes sans certificat client TLS. Tous les serveurs distants qui identifient Proofpoint par ce certificat ont refusé les e-mails. Les destinataires ordinaires utilisant TLS opportuniste ont continué à accepter les messages ; le problème n’est donc apparu que lorsque certains partenaires sont devenus injoignables.

Si vous utilisez PPS avec des correctifs automatiques et avez des partenaires qui exigent un certificat client, vérifiez ce paramètre après chaque correctif.

## Symptôme

Seules les connexions pour lesquelles le serveur distant vérifie le certificat client de Proofpoint sont concernées. Deux cas typiques :

- **Exchange Online avec un connecteur partenaire**, limité à un certificat par `RestrictDomainsToCertificate`. Le refus est `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost qui n’autorise le relais qu’avec un certificat client.** La réponse est `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Tous les autres destinataires continuent à recevoir leurs e-mails. La matrice de test présente donc une image incohérente : les e-mails externes sont transmis, mais les e-mails vers certains partenaires ou via un Smarthost donné échouent.

Le journal de messagerie de l’agent contient l’un des deux messages suivants pour chaque connexion sortante. Si le fichier est entièrement absent :

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Si le fichier est présent mais vide :

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Juste après, Sendmail établit tout de même la connexion avec TLS (`verify=OK`), mais sans certificat client. Pour le serveur distant, cela ressemble à un expéditeur inconnu.

## Cause

PPS associe un certificat à chaque service pour chaque serveur. Sous **System > Certificates > Services**, figurent notamment, pour chaque serveur, `SMTP Server`, `SMTP Client`, `Administration Server` et `End User Web Application / Encryption`. À partir de l’association pour `SMTP Client`, PPS génère le fichier `/etc/pki/tls/certs/sendmail-client.pem`, que Sendmail utilise comme certificat client. Si la valeur est `-none-`, PPS écrit un fichier vide ou ne l’écrit pas du tout.

Dans l’environnement étudié, l’association était définie sur `-none-` sur tous les serveurs après un correctif automatique. Sous **System > Config History**, le correctif était visible sous forme d’entrée `configuration changes for patch-…` créée par `system`; les fichiers sous `/etc/pki/tls/certs/` portaient le même horodatage. Les notes de version publiquement accessibles ne précisent pas si Proofpoint a introduit l’association avec ce correctif ou si celui-ci a réinitialisé une association existante. Je compléterai l’article dès que Proofpoint se prononcera à ce sujet.

Un retour en arrière sous **Config History** vers l’instantané du correctif n’a pas aidé. PPS a ensuite réécrit le fichier, mais avec 0 octet.

## Vérifier

Dans l’interface d’administration sur le serveur maître, sous **System > Certificates > Services**, sélectionnez successivement en haut, dans **Server**, chaque agent qui assure des livraisons sortantes, puis examinez la ligne `SMTP Client`.

Sur l’agent lui-même, deux commandes indiquent l’état :

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-l` | Format long avec droits, propriétaire, taille et date |
| `--time-style=full-iso` | Horodatage avec date, heure et fuseau horaire afin de le comparer à l’heure du correctif |
| `grep -E` | Expressions régulières étendues permettant de combiner les deux termes de recherche avec `\|` |
| `tail -5` | Afficher uniquement les cinq derniers résultats |

</details>

Si `sendmail-client.pem` est absent ou si le fichier fait 0 octet alors que `sendmail.pem` fait plusieurs milliers d’octets, aucun certificat n’est associé au service `SMTP Client`.

## Corriger

Sous **System > Certificates > Services**, sélectionnez un certificat valide pour `SMTP Client` sur chaque agent concerné, puis enregistrez avec **Save Changes**. Il s’agit généralement du même certificat public que celui défini pour `SMTP Server`.

La liste de sélection affiche tous les certificats importés, y compris les certificats expirés portant le même nom. Ils ne peuvent être distingués que par leur numéro de série entre parenthèses. Avant l’enregistrement, comparez le numéro de série avec la liste sous **System > Certificates > Certificates**, où figurent les dates d’émission et d’expiration.

Après l’enregistrement, PPS distribue la configuration aux agents. Vérification sur l’agent :

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `ls -l` | Affiche la taille et les droits du fichier ; après correction, il n’est plus vide |
| `grep -E 'STARTTLS=client\|stat='` | Filtre les messages TLS des connexions sortantes et le résultat de livraison |
| `tail -20` | Afficher uniquement les vingt dernières lignes |

</details>

Les messages `SSL_CTX_… failed` et `unsafe` ne doivent plus apparaître après l’enregistrement, et les livraisons aux partenaires doivent se terminer par `stat=Sent`. Sendmail livre lui-même depuis la file d’attente les e-mails précédemment reportés avec un code 4xx. Les e-mails refusés avec un code 5xx doivent être renvoyés par les expéditeurs.

Copier manuellement `sendmail.pem` vers `sendmail-client.pem` permet également de contourner la panne. PPS écrasera toutefois de nouveau le fichier lors du prochain déploiement de configuration tant que l’association manque dans l’interface.

## Contrôler après chaque correctif

Tant que la raison pour laquelle le correctif modifie l’association n’est pas clarifiée, je recommande de vérifier la page **Services** après chaque correctif. Une supervision qui alerte lors du message `sendmail-client.pem` dans le journal de messagerie des agents est plus simple. Sous **Licenses and Updates > Settings**, il est en outre possible de définir à quel moment les correctifs sont installés. Une fenêtre durant laquelle quelqu’un peut vérifier le flux de messagerie réduit considérablement la durée d’une telle panne.

## Client Authentication disparaît des certificats publics

Indépendamment du correctif, une deuxième évolution affecte précisément ces connexions. Jusqu’à présent, les certificats TLS publics contiennent généralement les deux usages dans l’extension `Extended Key Usage` : `TLS Web Server Authentication` et `TLS Web Client Authentication`. Les passerelles de messagerie utilisent donc leur certificat serveur également comme certificat client.

Le Chrome Root Program exige que les certificats publics délivrés à partir du 15.03.2027 ne contiennent plus que `serverAuth`. Depuis le 01.10.2025, DigiCert délivre par défaut des certificats TLS publics sans Client Authentication ; cet usage reste possible sur demande jusqu’au 01.03.2027. D’autres autorités de certification ont des calendriers comparables. OpenSSL indique si un certificat contient l’usage client :

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `x509` | Sous-commande pour afficher et traiter les certificats X.509 |
| `-in zertifikat.pem` | Fichier de certificat au format PEM |
| `-noout` | Ne pas réafficher le certificat lui-même |
| `-subject` | Afficher le titulaire (CN et autres champs) |
| `-enddate` | Afficher la date d’expiration |
| `-ext extendedKeyUsage` | Afficher uniquement l’extension contenant les usages |

</details>

Sous Windows, un double-clic sur le fichier montre la même information : onglet **Détails**, champ **Utilisation avancée de la clé**.

Les conséquences pour les connexions sortantes dépendent du serveur distant :

| Le serveur distant vérifie | Conséquence avec un certificat sans Client Authentication |
|---|---|
| Le nom dans le certificat (p. ex. Exchange Online via `TlsSenderCertificateName`) | Il n’est pas documenté si Exchange Online vérifie l’usage ; testez avant le changement de certificat |
| L’usage du certificat client | Refus, même si le nom et l’autorité de certification correspondent |
| L’empreinte du certificat client | Refus à chaque changement de certificat jusqu’à l’enregistrement de la nouvelle empreinte |

À cela s’ajoute la durée de validité plus courte des certificats publics. Selon la décision SC-081v3 du CA/Browser Forum, elle diminue progressivement de 398 jours à 47 jours d’ici mars 2029 ; la première étape, à 200 jours, s’applique depuis mars 2026. Chaque changement de certificat est un moment où un serveur distant peut ne pas connaître le nouveau certificat.

Pour les connexions dans lesquelles les partenaires identifient Proofpoint par un certificat client, je recommande donc :

1.  **Vérifier les certificats existants :** les certificats de passerelle actuels contiennent-ils encore l’usage client, et quand expirent-ils ?

2.  **Planifier un certificat client dédié**, qui contient durablement l’usage client, par exemple issu d’une PKI privée ou d’une offre de l’autorité de certification pour les connexions serveur à serveur. Les partenaires doivent le connaître.

3.  **Demander aux partenaires ce qu’ils vérifient :** nom, usage, émetteur ou empreinte. Cela détermine si un changement de certificat représente une modification pour eux.

4.  **Gérer le changement de certificat comme un changement**, avec une liste de tous les serveurs distants qui doivent connaître le certificat et un test pour chacun.

## Sources

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): paramètres `RestrictDomainsToCertificate` et `TlsSenderCertificateName` pour les connecteurs partenaires dans Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): section 1.3.2 sur les hiérarchies TLS dédiées aux serveurs ; les certificats à partir du 15.03.2027 ne contiennent plus que `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): absence de Client Authentication par défaut depuis le 01.10.2025, possible sur demande jusqu’au 01.03.2027, alternatives pour les connexions serveur à serveur.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): aperçu des échéances reportées pour Client Authentication et des paliers de durée de validité chez DigiCert et Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): décision concernant la réduction progressive de la durée de validité des certificats TLS publics jusqu’en mars 2029.
