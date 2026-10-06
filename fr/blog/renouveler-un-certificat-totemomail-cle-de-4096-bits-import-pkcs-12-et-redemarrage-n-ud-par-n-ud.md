---
title: "Renouveler un certificat Totemomail : clé de 4096 bits, import PKCS#12 et redémarrage nœud par nœud"
navTitle: "Renouveler le certificat"
description: "L’assistant de demande de Totemomail ne génère que des clés de 2048 bits sans noms alternatifs. Or, de nombreuses AC internes ne signent plus qu’en 4096 bits. La clé et la demande sont donc créées avec openssl, puis viennent la commande auprès de l’autorité PKI, l’import PKCS#12, l’association aux ports et le redémarrage de chaque nœud."
date: "2026-10-06"
kategorie: "Totemomail"
timeToRead: "12 min de lecture"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "renouveler-un-certificat-totemomail-cle-de-4096-bits-import-pkcs-12-et-redemarrage-n-ud-par-n-ud"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/fr/blog/renouveler-un-certificat-totemomail-cle-de-4096-bits-import-pkcs-12-et-redemarrage-n-ud-par-n-ud
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:32:07.272Z
translationReview: automatic
---

# Renouveler un certificat Totemomail : clé de 4096 bits, import PKCS#12 et redémarrage nœud par nœud

Renouveler le certificat serveur d’un cluster Totemomail (désormais Kiteworks Email Protection Gateway) semble être une tâche routinière : générer une demande dans l’interface d’administration, la faire signer, importer la réponse. En pratique, cette méthode échoue souvent dès la première étape. La boîte de dialogue « New PKCS#10 » génère systématiquement une clé de 2048 bits et ne propose aucun champ pour les noms alternatifs (Subject Alternative Names). De nombreuses autorités de certification internes ne signent désormais plus qu’en 4096 bits, et les homologues TLS actuels ne reconnaissent même pas un certificat sans noms alternatifs comme valide pour un nom.

La procédure suivante a fait ses preuves lors d’un remplacement à l’automne 2026 en préproduction et en production : clé et demande avec openssl sur la passerelle, commande auprès de l’autorité PKI, import au format PKCS#12, association aux ports et redémarrage nœud par nœud. Plusieurs particularités sont apparues lors du remplacement, notamment une erreur de l’interface qui survient lors du détachement de l’ancien certificat.

## Deux certificats à des fins différentes

Un cluster Totemomail derrière Exchange Online nécessite en règle générale deux types de certificats.

| Type | Usage | Émetteur | Durée de validité |
|---|---|---|---|
| Interne | Interface web, administration, connexions SMTP internes. Contient les noms internes des nœuds. | PKI interne, par exemple Active Directory Certificate Services | au choix, typiquement 12 à 13 mois |
| Public | Liaison entre Exchange Online et la passerelle, lorsque Exchange Online doit vérifier le certificat | autorité de certification publique | 200 jours maximum depuis le 15.03.2026, 100 jours maximum à partir du 15.03.2027 |

Un seul certificat ne fonctionne pas pour les deux usages. Les autorités de certification publiques n’émettent ni noms internes ni noms courts sans domaine, et chaque certificat public apparaît dans les journaux Certificate Transparency. À l’inverse, Exchange Online ne fait pas confiance à une chaîne interne. L’article [Boucle de messagerie avec une passerelle de chiffrement derrière EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online) décrit la manière dont Exchange Online traite la liaison vers une passerelle de chiffrement.

Les étapes suivantes concernent le certificat interne. Pour le certificat public, la procédure est identique jusqu’à l’import ; les différences figurent dans la dernière section.

## Vue d’ensemble de la procédure

1. Générer la clé et la demande avec openssl sur un nœud.
2. Commander la demande auprès de l’autorité PKI.
3. Vérifier le certificat livré.
4. Regrouper le certificat, la clé et l’intermédiaire dans un fichier PKCS#12 et le récupérer sur l’ordinateur équipé du navigateur.
5. Importer le certificat dans l’interface Totemomail et l’associer aux ports.
6. Redémarrer et vérifier chaque nœud individuellement.
7. Tester le flux de messagerie, puis nettoyer.

Prévoyez au moins trois semaines avant l’expiration. Tant que l’ancien certificat est valide, un retour arrière reste possible. Si votre environnement dispose d’une préproduction, effectuez d’abord le remplacement à cet endroit et utilisez cette exécution comme référence.

## Étape 1 : clé et demande avec openssl

Travaillez sur un nœud du cluster avec votre utilisateur personnel, et non avec le compte de service `totemo`. Vous pourrez ainsi récupérer directement le fichier PKCS#12 plus tard via `scp`. Aucun droit root n’est requis pour toutes les étapes.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

La configuration contient le titulaire, les usages et tous les noms alternatifs. Les noms de l’exemple sont des espaces réservés : trois nœuds et les noms de service sous lesquels le cluster est joint en interne.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Options expliquées</summary>

| Entrée | Effet |
|---|---|
| `default_md = sha256` | Algorithme de hachage pour la signature de la demande |
| `prompt = no` | Reprendre les valeurs du fichier au lieu de les demander de manière interactive |
| `req_extensions = ext` | Écrire les extensions de la section `[ ext ]` dans la demande |
| `basicConstraints = critical, CA:FALSE` | Certificat final, pas une autorité de certification |
| `keyUsage` | Clé destinée à la signature et à l’échange de clés, comme habituellement pour les serveurs TLS |
| `extendedKeyUsage = serverAuth, clientAuth` | Authentification serveur et client. La passerelle est serveur sur certaines liaisons et cliente sur d’autres. |
| `subjectAltName = @alt` | Noms alternatifs de la section `[ alt ]` |

</details>

N’incluez que des noms complets. De nombreuses autorités d’enregistrement refusent les noms courts sans domaine, et vous ne devez pas compter sur leur présence dans le certificat.

Générez la clé sans phrase de passe et protégez-la au moyen des droits de fichier. Elle ne reste sur le nœud que jusqu’à l’import, puis elle est supprimée. Une phrase de passe sur une clé qui n’existe que quelques jours apporte peu de protection et crée un nouveau risque : si elle est perdue, la clé devient inutilisable et le certificat doit être réémis.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `genpkey -algorithm RSA` | Générer une nouvelle clé privée de type RSA |
| `-pkeyopt rsa_keygen_bits:4096` | Longueur de clé de 4096 bits |
| `-out intern.key` | Fichier de la clé, sans phrase de passe |
| `req -new` | Générer une nouvelle demande de certificat (CSR) |
| `-key intern.key` | Utiliser une clé existante |
| `-config intern.cnf` | Titulaire, extensions et noms provenant de la configuration |
| `-out intern.csr` | Fichier de la demande |

</details>

Vérifiez la demande avant qu’elle ne quitte la machine :

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Vous devez obtenir `verify OK`, le bon titulaire, `4096 bit` et le nombre de vos noms. Notez également l’empreinte de la clé publique. Elle vous permettra ensuite d’attribuer sans ambiguïté le certificat livré à cette clé, même si deux demandes portent le même titulaire :

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Étape 2 : commande auprès de l’autorité PKI

L’autorité qui émet le certificat a besoin des informations suivantes pour chaque demande :

- le CSR sous forme de texte
- l’empreinte SHA-256 de la clé publique
- le type : interne ou public, et lequel en cas de plusieurs environnements
- la liste des noms à copier, un nom par ligne
- longueur de clé 4096, usages étendus Server Authentication et Client Authentication
- la date à laquelle le certificat doit être disponible, au moins deux semaines avant l’expiration

Lors d’un remplacement dans plusieurs environnements et avec plusieurs types de certificats, un aperçu au début de l’e-mail est utile : quels certificats existent, lesquels sont émis en interne et lesquels publiquement, et dans quel ordre ils sont nécessaires. Aucun changement sur le système n’est requis pour l’émission. Les certificats de tous les environnements peuvent donc être émis simultanément, même si leur déploiement s’effectue successivement.

Lors du remplacement à l’automne 2026, plusieurs points concernant l’autorité d’enregistrement (RA) ont été relevés, et ils devraient se retrouver de manière similaire dans de nombreux environnements :

- **La RA définit elle-même le titulaire.** Le certificat ne contenait que le pays, l’organisation et le Common Name, même si la demande incluait l’unité organisationnelle, le lieu et le canton.
- **Pas de noms courts.** Les noms sans domaine étaient absents du certificat émis.
- **Noms repris manuellement.** La RA n’a pas repris les noms alternatifs du CSR ; ils ont été saisis dans le formulaire. Un nom est arrivé tronqué. La liste de noms livrée doit donc toujours être vérifiée.
- **Mauvais profil.** Une RA qui gère à la fois la CA interne et une CA publique a d’abord émis les demandes de certificats publics au moyen du profil interne. L’émetteur indiquait la CA interne. De tels certificats ne servent à rien pour la liaison vers Exchange Online.
- **Un seul nom pour les certificats publics individuels.** Un produit Single-Domain échoue avec une demande comportant deux noms, par exemple avec « Only one Subject Alternative Name is allowed ». Un nom suffit toutefois, voir la section sur le certificat public.

Les certificats résultant d’émissions erronées devraient ensuite être révoqués par l’autorité PKI. Ils contiennent une clé valide et restent sinon valables pendant un an.

## Étape 3 : vérifier la livraison

Placez le certificat livré dans le même répertoire que la clé, par exemple avec `cat > intern.crt`, collez le contenu, puis `Strg+D`. Vérifiez ensuite :

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Contrôlez l’émetteur, la liste complète des noms sans fautes de frappe ni doublons, ainsi que les deux usages. La comparaison des empreintes montre si le certificat et la clé correspondent. Les deux valeurs doivent être identiques :

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

Pour un certificat public, deux contrôles supplémentaires s’ajoutent. L’OID de stratégie `2.23.140.1.2.2` désigne un certificat validé par l’organisation selon les règles du CA/Browser Forum, et le certificat doit contenir des preuves Certificate Transparency intégrées. Quelques minutes après son émission, il apparaît sous son nom sur crt.sh. Si l’un ou l’autre manque, ce n’est pas un certificat public, quelle que soit l’étiquette de la livraison.

## Étape 4 : créer le fichier PKCS#12

Totemomail importe ensemble le certificat et la clé au format PKCS#12. Récupérez pour cela le certificat de l’intermédiaire émetteur. Son adresse figure dans le certificat sous `Authority Information Access` :

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

Si `file` n’indique pas `PEM certificate`, le fichier est encodé en DER et doit être converti :

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

Le Subject Key Identifier de l’intermédiaire doit correspondre à l’Authority Key Identifier du certificat. Créez ensuite le fichier. La commande demande un mot de passe d’exportation dont vous aurez besoin lors de l’import :

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-export` | Créer un fichier PKCS#12 |
| `-inkey intern.key` | Clé privée |
| `-in intern.crt` | Certificat émis |
| `-certfile issuing.crt` | Autres certificats de la chaîne, ici l’intermédiaire |
| `-name "SecureMail"` | Nom d’affichage de l’entrée dans le fichier |
| `-out intern.p12` | Fichier cible, protégé par le mot de passe d’exportation |

</details>

Vérification : deux entrées doivent apparaître, le certificat et l’intermédiaire :

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

L’interface Totemomail fonctionne dans le navigateur, généralement sur un jumphost. Le fichier doit y être transféré. Sous Windows, le client OpenSSH avec `scp` est disponible :

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

Si vous avez tout de même généré la clé sous `totemo`, elle se trouve sous `/opt/totemomail`, et votre utilisateur ne peut pas la lire. Copiez alors brièvement le fichier PKCS#12 vers `/tmp`, récupérez-le depuis cet emplacement et supprimez-le immédiatement. Le détour via le presse-papiers avec Base64 fonctionne, mais il est sujet aux erreurs pour une ligne d’environ 10'000 caractères.

## Étape 5 : importer et associer dans Totemomail

Sous `Key Management` :

1. **`Issuer Certificates`** : importez l’intermédiaire `issuing.crt`.
2. **`Own Server Certificates`**, bouton **`Import`** : la boîte de dialogue propose deux méthodes. À gauche, « Import certificate » est prévu pour un certificat avec clé, donc le fichier PKCS#12. À droite, « Import a PKCS#10 certificate reply » ne sert qu’aux réponses aux demandes que Totemomail a lui-même générées. Choisissez la gauche et saisissez le mot de passe d’exportation à la deuxième étape.
3. Ouvrez le nouveau certificat (icône crayon). Sous `Connector`, tous les ports doivent être cochés ; dans l’installation décrite, `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` et `8444=AdminAPI`. `Host` est réglé sur `*`.
4. Ouvrez l’ancien certificat et décochez tous les ports sauf un.

Le point 4 correspond à l’erreur apparue lors du remplacement : si tous les ports sont décochés sur l’ancien certificat, l’interface affiche « Could not edit selected server certificate ». Le journal contient :

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

Totemomail ne tolère pas de certificat serveur sans port. Laissez donc un port, par exemple `8444=AdminAPI`, puis supprimez l’ancien certificat après la période d’observation. La suppression fonctionne au lieu du décochage ; le certificat se retrouve sous `Deleted Certificates`.

Trois observations sont utiles pour comprendre l’association :

- **La liste des ports ne s’applique qu’aux services web.** Le port 25 n’y apparaît pas. Si aucun certificat de type SMTPS n’est présent, SMTP utilise le certificat HTTPS. C’est pourquoi le port 25 présente le nouveau certificat après le remplacement, même si SMTPS n’est pas coché dans la liste.
- **Une association explicite prévaut sur l’astérisque.** Le port conservé sur l’ancien certificat continue de présenter l’ancien, même si le nouveau est enregistré avec `*` pour tous.
- **La liste est commune au cluster.** Elle est identique sur chaque nœud, indépendamment du fait que le nœud ait déjà repris la modification. Seule une interrogation des ports indique ce qu’un nœud présente réellement.

## Étape 6 : redémarrage nœud par nœud

Totemomail charge les certificats au démarrage. Après l’import, tous les nœuds continuent de présenter l’ancien certificat jusqu’à leur redémarrage. Redémarrez chaque nœud individuellement, jamais tous simultanément, afin que des nœuds restent toujours actifs derrière l’équilibreur de charge. Commencez par les nœuds auxquels vous n’êtes pas connecté et prenez le nœud avec l’interface ouverte en dernier.

Sur le nœud, sous `totemo` :

```bash
totemomail stop
totemomail start
```

Vous trouverez d’autres indications sur l’arrêt contrôlé dans l’article [Principaux contrôles pour les administrateurs Totemomail](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

Les services web sont de nouveau accessibles après environ une minute, mais le service SMTP sur le port 25 seulement après quelques minutes. Une réponse vide sur le port 25 peu après le démarrage ne signale donc pas encore une erreur. Passez au nœud suivant uniquement lorsqu’un nœud a basculé sur tous les ports.

La vérification sur tous les nœuds et ports :

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `s_client -connect host:port` | Établir une connexion TLS au service |
| `-starttls smtp` | Sur le port 25, effectuer d’abord le dialogue SMTP, puis basculer vers TLS avec STARTTLS |
| `x509 -noout` | Lire le certificat sans l’afficher |
| `-serial -enddate` | Afficher le numéro de série et la date d’expiration |

</details>

Si le port 25 reste vide après cinq minutes, `ss -lnt | grep ':25 '` indique si le service est à l’écoute, et le journal sous `/opt/totemomail` en donne la raison.

L’option `-showcerts` indique si Totemomail transmet l’intermédiaire :

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

Lors du remplacement décrit, seul le certificat final est apparu, bien que l’intermédiaire figurât dans le fichier PKCS#12 et sous `Issuer Certificates`. Cela n’a aucune conséquence pour les liaisons internes où personne ne vérifie le certificat. En revanche, la chaîne doit être complète pour la vérification par Exchange Online.

## Étape 7 : tests, retour arrière, nettoyage

Après le dernier redémarrage, testez le flux de messagerie dans les deux sens : un message provenant de l’extérieur à travers la boucle et un message sortant via la passerelle. Dans le suivi des messages d’Exchange Online, les deux segments doivent apparaître comme remis, et rien ne doit rester bloqué dans la file d’attente en direction de la passerelle.

Le **retour arrière** consiste à recocher les ports de l’ancien certificat, à décocher tous les ports du nouveau sauf un, puis à redémarrer de nouveau les nœuds individuellement. Cela n’est possible que tant que l’ancien certificat reste valide.

Après des tests réussis :

- supprimer le fichier PKCS#12 du jumphost
- supprimer le répertoire de travail sur le nœud ; la clé se trouve désormais dans le magasin de clés de Totemomail
- après quelques jours, supprimer l’ancien certificat et redémarrer une nouvelle fois les nœuds individuellement afin que le dernier port bascule également
- faire révoquer les émissions erronées auprès de l’autorité PKI
- inscrire la nouvelle date d’expiration dans le système de suivi

Annoncez le remplacement aux administrateurs si le nouveau certificat ne contient plus de noms courts. Les personnes qui appelaient jusqu’ici l’interface avec `https://gw01:8443` verront ensuite un avertissement de certificat. Cela vaut également pour les systèmes de supervision et les scripts qui utilisent des noms courts.

## Particularités en bref

| Observation | Conséquence | Gestion |
|---|---|---|
| « New PKCS#10 » génère 2048 bits sans noms alternatifs | Demande inutilisable pour des CA en 4096 bits | Générer la clé et la demande avec openssl |
| Les modifications ne prennent effet qu’après le redémarrage | Les nœuds continuent de présenter l’ancien certificat | Redémarrer chaque nœud individuellement |
| Le port 25 devient disponible quelques minutes après les services web | Réponse vide peu après le démarrage | Attendre avant de passer au nœud suivant |
| Un certificat sans port déclenche une NullPointerException | L’ancien certificat ne peut pas être entièrement détaché | Laisser un port, puis supprimer ultérieurement |
| Une association explicite prévaut sur `*` | Un port continue de présenter l’ancien certificat | Supprimer l’ancien certificat après la période d’observation |
| Seul le certificat final est envoyé | Les homologues qui vérifient ne peuvent pas constituer la chaîne | Clarifier avant une vérification par Exchange Online |
| La RA reprend les noms manuellement | Risque de fautes de frappe et de noms manquants | Vérifier la liste des noms livrée |

## Le certificat public pour la liaison vers Exchange Online

Pour qu’Exchange Online vérifie la liaison vers la passerelle en plus de la chiffrer, la passerelle a besoin d’un certificat public sur le port 25. Ce n’est qu’alors que le connecteur sortant peut être configuré avec `TlsSettings DomainValidation` et un `TlsDomain`, et que le connecteur entrant peut être associé au certificat via `TlsSenderCertificateName`.

Un seul nom suffit à cette fin. Exchange Online compare uniquement, dans les deux sens, le nom figurant dans le certificat avec la valeur configurée ; il ne vérifie pas l’adresse qui se trouve derrière. Un certificat pour le nom sous lequel la passerelle est connue à l’extérieur couvre l’aller et le retour. Lors de la commande, demandez explicitement Client Authentication : plusieurs autorités de certification publiques ont supprimé cet usage des certificats TLS en 2026. Il est nécessaire pour le retour, où la passerelle s’identifie comme cliente.

Pour une utilisation dans Totemomail, les observations ci-dessus suggèrent la méthode suivante : importer le certificat public comme type SMTPS afin que le port 25 le présente et que les services web conservent le certificat interne. Deux points doivent auparavant être clarifiés. La chaîne doit être envoyée intégralement, et le port 25 présente ensuite le certificat public à tous les expéditeurs, y compris internes, tel qu’une passerelle en amont. Si un expéditeur attend un certificat précis, cette liaison échoue.

## Sources

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/) : calendrier pour la durée de validité des certificats TLS publics, 200 jours à partir de mars 2026, 100 jours à partir de mars 2027, 47 jours à partir de mars 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector) : paramètres `TlsSettings` et `TlsDomain` pour vérifier le certificat du correspondant.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector) : paramètre `TlsSenderCertificateName` pour l’association via le certificat de l’expéditeur.

4.  [Documentation OpenSSL : openssl-req](https://docs.openssl.org/master/man1/openssl-req/) : structure du fichier de configuration et options pour les demandes de certificat.

5.  [Documentation OpenSSL : openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/) : création et vérification de fichiers PKCS#12.

6.  [crt.sh](https://crt.sh) : recherche dans les journaux Certificate Transparency pour vérifier l’émission d’un certificat public.
