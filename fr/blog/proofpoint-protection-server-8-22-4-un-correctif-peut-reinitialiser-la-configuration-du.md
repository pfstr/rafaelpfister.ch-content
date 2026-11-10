---
title: "Proofpoint Protection Server 8.22.4 : le patch 5807 laisse le client SMTP sans certificat"
navTitle: "PPS 8.22.4 certificat"
description: "Les patchs 5806 et 5807 introduisent dans Proofpoint Protection Server 8.22.4, sous System > Certificates > Services, une attribution distincte « SMTP Client ». Si elle est définie sur « -none- », Sendmail délivre sans certificat client TLS. Les systèmes distants qui vérifient les certificats refusent alors, par exemple Exchange Online avec 5.7.51 ou un Smarthost avec « Relay access denied ». Voici comment vérifier et corriger le problème."
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
translationSourceHash: fcb36b2ae8af7420156aca54a99d61f4702aeb47781e1081e27edf7419e77db8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:20:31.490Z
translationReview: automatic
aktualisiert: 2026-11-06
url: https://rafaelpfister.ch/fr/blog/proofpoint-protection-server-8-22-4-un-correctif-peut-reinitialiser-la-configuration-du
---

J’ai aujourd’hui analysé une panne du flux de messagerie sortant d’un Proofpoint Protection Server (PPS) 8.22.4. Conclusion rapide : après l’installation automatique d’un patch, l’attribution **SMTP Client** sous **System > Certificates > Services** était définie sur `-none-`. Sendmail a alors établi des connexions sortantes sans certificat client TLS. Tous les systèmes distants qui identifient Proofpoint au moyen de ce certificat ont rejeté les e-mails. Les destinataires ordinaires avec TLS opportuniste ont continué à les accepter ; le problème n’est donc apparu que lorsque certains partenaires sont devenus injoignables.

Si vous exploitez PPS avec des patchs automatiques et avez des partenaires qui exigent un certificat client, vérifiez ce paramètre après chaque patch.

## Symptôme

Seules les connexions pour lesquelles le système distant vérifie le certificat client de Proofpoint sont concernées. Deux cas typiques :

- **Exchange Online avec un connecteur partenaire**, limité à un certificat via `RestrictDomainsToCertificate`. Le rejet est `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost qui n’autorise le relais qu’avec un certificat client.** La réponse est `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Tous les autres destinataires continuent à recevoir leurs e-mails. La matrice de tests présente donc un tableau hétérogène : les e-mails externes passent, mais ceux destinés à certains partenaires ou envoyés via un Smarthost donné échouent.

Le journal de messagerie de l’agent contient l’un des deux messages suivants pour chaque connexion sortante. Si le fichier est totalement absent :

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Si le fichier existe mais est vide :

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Juste après, Sendmail établit tout de même la connexion avec TLS (`verify=OK`), mais sans certificat client. Pour le système distant, cela ressemble à un expéditeur inconnu.

## Cause

PPS attribue un certificat à chaque service sur chaque serveur. Sous **System > Certificates > Services**, chaque serveur affiche notamment `SMTP Server`, `SMTP Client`, `Administration Server` et `End User Web Application / Encryption`. À partir de l’attribution de `SMTP Client`, PPS génère le fichier `/etc/pki/tls/certs/sendmail-client.pem`, que Sendmail utilise comme certificat client. Si `-none-` y est défini, PPS écrit le fichier vide ou ne l’écrit pas du tout.

L’attribution `SMTP Client` n’existe dans la version 8.22.4 que depuis les patchs 5806 et 5807, publiés par Proofpoint le 29.10.2026. Les deux portent le titre « TLS client and server certificate authentication improvement » et corrigent, selon la Knowledge Base, l’impossibilité pour PPS 8.22.4 d’attribuer des certificats séparés au serveur SMTP et au client SMTP. Dans la version 8.23.0, la fonctionnalité est intégrée de manière standard (Release Notes, section « SMTP Client Certificate »).

| Patch | Contenu selon la Knowledge Base |
|---|---|
| 5806 | Met notamment à jour `SysCertificates.pm`, `Certificate.pm`, `TLS.pm`, `pp-sendmail.pl` et le modèle `sendmail.mc`, redémarre l’interface d’administration (`admind`) et met à jour le RPM `pps-pfpt` |
| 5807 | Nécessite 5806, ne modifie aucun fichier et ne redémarre aucun processus ; applique uniquement les modifications de configuration nécessaires à la séparation des certificats serveur et client |

Proofpoint a donc introduit cette attribution, plutôt que de réinitialiser une attribution existante. Le patch ne reprend pas de certificat : après l’installation, la nouvelle ligne est définie sur `-none-`, et Sendmail n’utilise dès lors plus de certificat client. Les descriptions des patchs ne le signalent pas.

Dans l’environnement étudié, l’attribution était ensuite définie sur `-none-` sur tous les serveurs. Sous **System > Config History**, le patch 5807 était visible comme entrée `configuration changes for patch-…` avec le créateur `system`, et les fichiers sous `/etc/pki/tls/certs/` portaient le même horodatage. En revanche, 5807 n’apparaissait pas dans **Software Patch History**. Cela correspond au fait que 5807 n’installe aucun logiciel, mais modifie uniquement la configuration. **Config History** indique donc le moment du changement, tandis que la modification logicielle proprement dite relève du patch 5806.

Un retour en arrière sous **Config History** vers le snapshot du patch n’a pas aidé. PPS a ensuite réécrit le fichier, mais avec une taille de 0 octet.

## Vérifier

Dans l’interface d’administration du maître, sous **System > Certificates > Services**, sélectionnez successivement en haut, dans **Server**, chaque agent qui délivre des e-mails vers l’extérieur, puis examinez la ligne `SMTP Client`.

Sur l’agent lui-même, deux commandes indiquent l’état :

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-l` | Format long avec droits, propriétaire, taille et date |
| `--time-style=full-iso` | Horodatage avec date, heure et fuseau horaire, pour le comparer à l’heure du patch |
| `grep -E` | Expressions régulières étendues afin de pouvoir combiner les deux termes de recherche avec `\|` |
| `tail -5` | Afficher uniquement les cinq derniers résultats |

</details>

Si `sendmail-client.pem` est absent ou si le fichier fait 0 octet, alors que `sendmail.pem` fait plusieurs milliers d’octets, aucun certificat n’est attribué au service `SMTP Client`.

## Corriger

Sous **System > Certificates > Services**, sélectionnez un certificat valide pour `SMTP Client` sur chaque agent concerné, puis enregistrez avec **Save Changes**. En règle générale, il s’agit du même certificat public que celui indiqué pour `SMTP Server`.

La liste de sélection peut contenir plusieurs certificats portant le même nom, y compris des certificats expirés. Ils ne se distinguent que par le numéro de série entre parenthèses. Avant l’enregistrement, comparez le numéro de série avec la liste sous **System > Certificates > Certificates**, où figurent les dates d’émission et d’expiration.

La liste n’affiche pas nécessairement tous les certificats importés pour `SMTP Client`. Dans l’environnement étudié, certains certificats étaient absents bien qu’ils soient valides et que les exemplaires expirés aient déjà été supprimés. Ni les articles sur les patchs ni les Release Notes 8.23 ne décrivent les critères appliqués par PPS pour filtrer la liste. Si un certificat est absent, vérifiez deux points :

- **Clé privée :** si le certificat a été importé comme PEM sans clé, PPS ne peut l’attribuer à aucun service. Les fichiers PKCS12 contiennent toujours la clé.
- **Usage prévu :** le certificat contient-il, dans `Extended Key Usage`, l’usage `TLS Web Client Authentication` ? De nombreux certificats publics émis depuis l’automne 2025 ne le contiennent plus (voir ci-dessous). Pour le service `SMTP Client`, un certificat dépourvu de cet usage n’est de toute façon pas approprié.

Dès que Proofpoint confirmera les critères, je les ajouterai ici.

Après l’enregistrement, PPS distribue la configuration aux agents. Vérification sur l’agent :

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `ls -l` | Affiche la taille et les droits du fichier ; après correction, il n’est plus vide |
| `grep -E 'STARTTLS=client\|stat='` | Filtre les messages TLS des connexions sortantes et le résultat de remise |
| `tail -20` | Afficher uniquement les vingt dernières lignes |

</details>

Les messages `SSL_CTX_… failed` et `unsafe` ne doivent plus apparaître après l’enregistrement, et les remises aux partenaires doivent se terminer par `stat=Sent`. Les e-mails précédemment mis en attente avec un code 4xx sont remis par Sendmail depuis la file d’attente. Les e-mails rejetés avec un code 5xx doivent être renvoyés par les expéditeurs.

Copier manuellement `sendmail.pem` vers `sendmail-client.pem` permet également de contourner temporairement la panne. PPS écrase toutefois à nouveau le fichier lors du prochain push de configuration tant que l’attribution manque dans l’interface.

## Contrôler après chaque patch

Le patch 5807 a modifié la configuration sans que sa description n’indique les conséquences. Je recommande donc de contrôler la page **Services** après chaque patch et de consulter **Config History** pour savoir si le patch a apporté des modifications de configuration. Si le patch 5807 n’est pas encore installé sur un système 8.22.4, attribuez directement un certificat à `SMTP Client` après l’installation. Une surveillance qui alerte sur le message `sendmail-client.pem` dans le journal de messagerie des agents est plus simple. Sous **Licenses and Updates > Settings**, il est aussi possible de définir quand les patchs sont installés. Une fenêtre durant laquelle quelqu’un peut vérifier le flux de messagerie réduit fortement ce type de panne.

## Client Authentication disparaît des certificats publics

Indépendamment du patch, une seconde modification affecte précisément ces connexions. Jusqu’à présent, les certificats TLS publics contenaient généralement les deux usages dans l’extension `Extended Key Usage` : `TLS Web Server Authentication` et `TLS Web Client Authentication`. Les passerelles de messagerie utilisent donc aussi leur certificat serveur comme certificat client.

Le Chrome Root Program exige que les certificats publics émis à partir du 15.03.2027 ne contiennent plus que `serverAuth`. DigiCert émet par défaut des certificats TLS publics sans Client Authentication depuis le 01.10.2025 ; sur demande, cet usage reste possible jusqu’au 01.03.2027. D’autres AC suivent des calendriers comparables. OpenSSL indique si un certificat contient l’usage client :

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `x509` | Sous-commande servant à afficher et modifier des certificats X.509 |
| `-in zertifikat.pem` | Fichier de certificat au format PEM |
| `-noout` | Ne pas afficher à nouveau le certificat lui-même |
| `-subject` | Afficher le titulaire (CN et autres champs) |
| `-enddate` | Afficher la date d’expiration |
| `-ext extendedKeyUsage` | Afficher uniquement l’extension contenant les usages |

</details>

Sous Windows, un double-clic sur le fichier affiche la même information : onglet **Details**, champ **Enhanced Key Usage**.

Les conséquences pour les connexions sortantes dépendent du système distant :

| Le système distant vérifie | Conséquence avec un certificat sans Client Authentication |
|---|---|
| Le nom dans le certificat (p. ex. Exchange Online via `TlsSenderCertificateName`) | Il n’est pas documenté qu’Exchange Online vérifie l’usage prévu ; testez avant de changer de certificat |
| L’usage prévu du certificat client | Rejet, même si le nom et l’AC correspondent |
| L’empreinte du certificat client | Rejet à chaque changement de certificat jusqu’à ce que la nouvelle empreinte soit enregistrée |

À cela s’ajoute la durée de validité plus courte des certificats publics. Après la décision SC-081v3 du CA/Browser Forum, elle diminue progressivement de 398 jours à 47 jours d’ici mars 2029 ; le premier palier, à 200 jours, s’applique depuis mars 2026. Chaque changement de certificat est un moment où un système distant peut ne pas connaître le nouveau certificat.

Pour les connexions dans lesquelles des partenaires identifient Proofpoint par un certificat client, je recommande donc :

1.  **Vérifier les certificats existants :** les certificats actuels de la passerelle contiennent-ils encore l’usage client et quand expirent-ils ?

2.  **Prévoir un certificat client dédié**, contenant durablement l’usage client, par exemple issu d’une PKI privée ou d’une offre de l’AC pour les connexions serveur à serveur. Les partenaires doivent le connaître.

3.  **Demander aux partenaires ce qu’ils vérifient :** nom, usage prévu, émetteur ou empreinte. Cela détermine si un changement de certificat représente une modification pour eux.

4.  **Gérer les changements de certificat comme des changements**, avec une liste de tous les systèmes distants qui doivent connaître le certificat et un test par système distant.

## Sources

1.  [Patch 5806 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, connexion requise)](https://proofpoint.my.site.com/community/s/article/Patch-5806---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): article 000027579 du 29.10.2026, fichiers modifiés et redémarrage de l’interface d’administration.

2.  [Patch 5807 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, connexion requise)](https://proofpoint.my.site.com/community/s/article/Patch-5807---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): article 000027580 du 29.10.2026, modification de configuration pure avec dépendance à 5806.

3.  [Proofpoint Enterprise Protection 8.23.X Release Notes (Proofpoint Knowledge Base, connexion requise)](https://proofpoint.my.site.com/community/s/article/Email-Protection-PPS-PoD-Proofpoint-Enterprise-Protection-8-23-X-Release-Notes): article 000027117, section « SMTP Client Certificate » à la page 5 de la révision A.

4.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): paramètres `RestrictDomainsToCertificate` et `TlsSenderCertificateName` pour les connecteurs partenaires dans Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): section 1.3.2 sur les hiérarchies TLS serveur dédiées ; les certificats émis à partir du 15.03.2027 ne contiennent plus que `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): sans Client Authentication par défaut depuis le 01.10.2025, sur demande jusqu’au 01.03.2027, alternatives pour les connexions serveur à serveur.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): aperçu des échéances reportées pour Client Authentication et des paliers de validité chez DigiCert et Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): décision relative à la réduction progressive de la durée de validité des certificats TLS publics jusqu’en mars 2029.
