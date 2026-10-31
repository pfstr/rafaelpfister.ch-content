---
title: "Guide pour les administrateurs DNS : MX, SPF, DKIM, DMARC et les sources d’erreur habituelles"
navTitle: "Enregistrements DNS de messagerie"
description: "Les personnes qui gèrent une zone reçoivent généralement les enregistrements de messagerie déjà prêts et n’ont plus qu’à les publier. Les problèmes récurrents : la limite de 255 octets pour DKIM, les enregistrements SPF en double, la limite de recherches, un MX pointant vers un CNAME, le suffixe de zone ajouté automatiquement et des politiques que plus personne n’applique."
date: "2026-08-04"
kategorie: "SMTP et flux de messagerie"
timeToRead: "15 min de lecture"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - smtp-verbindung-testen-linux
  - ghost-sender-exchange-online-nebeneingang
slug: "guide-pour-administrateurs-dns-mx-spf-dkim-dmarc-et-les-pieges-courants"
translationId: "article-e4699ad7fcea2e20"
translationOf: dns-records-e-mail-stolpersteine
translationSourceHash: e78b63f0179944cf4c36caf7aabfb9ceb0c5dd40d2d3cf1a25e92fee8cbacaca
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:41:18.387Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/guide-pour-administrateurs-dns-mx-spf-dkim-dmarc-et-les-pieges-courants
---

# Guide pour les administrateurs DNS : MX, SPF, DKIM, DMARC et les sources d’erreur habituelles

Lorsqu’on gère une zone DNS, on reçoit rarement les enregistrements de messagerie rédigés par soi-même. L’équipe mail, un fournisseur ou un prestataire marketing envoie une ligne en précisant qu’elle doit « seulement être publiée ». C’est précisément là que surviennent la plupart des erreurs, car les enregistrements de messagerie sont le type d’enregistrement pour lequel une faute de frappe peut avoir deux conséquences très différentes. Soit la livraison s’interrompt immédiatement et quelqu’un le signale en quelques minutes, soit elle continue sans changement et seule l’authentification de l’expéditeur échoue silencieusement. Le second cas passe régulièrement inaperçu pendant des mois, jusqu’à ce qu’un grand destinataire place le domaine en quarantaine.

Depuis que Google et Yahoo ont renforcé leurs exigences pour les expéditeurs de masse en février 2024 et que Microsoft leur a emboîté le pas en mai 2025, la tolérance envers les domaines partiellement configurés est devenue faible. SPF, DKIM et un enregistrement DMARC ne sont plus un bonus pour les expéditeurs dépassant un certain volume, mais une condition de livraison.

Tous les exemples de cet article utilisent `example.com` et des sélecteurs génériques. Les valeurs affichées sont abrégées afin de rester lisibles.

## Règles valables pour chaque enregistrement de messagerie

### La limite de 255 octets des enregistrements TXT

Selon la RFC 1035, un enregistrement TXT se compose d’une ou plusieurs `character-strings`, et chacune de ces chaînes de caractères peut contenir au maximum 255 octets. L’enregistrement dans son ensemble peut être plus long, mais il doit alors être divisé en plusieurs chaînes. Les systèmes qui l’évaluent concatènent ces parties sans séparateur.

Cela devient concrètement pertinent à un endroit précis : avec les clés DKIM de 2048 bits. Leur valeur Base64 compte environ 400 caractères et ne tient pas dans une seule chaîne.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La plupart des systèmes de gestion DNS effectuent eux-mêmes cette répartition lorsque la valeur est saisie dans le champ normal. En revanche, quiconque ajoute manuellement des guillemets doit respecter exactement cette limite. Une valeur coupée avec un espace à la jonction produit une clé qui existe syntaxiquement mais qui ne correspond plus sur le plan cryptographique.

Il est important de vérifier ensuite, car une clé mal assemblée semble parfaitement normale dans l’interface graphique :

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT selector1._domainkey.example.com` | Type d’enregistrement et nom de l’enregistrement de clé DKIM |
| `tr -d '" '` | Supprime les guillemets et les espaces, et recompose donc les chaînes partielles comme un vérificateur les lit |
| `wc -c` | Compte les caractères de la valeur reconstituée ; la longueur doit correspondre au modèle |

</details>

### Un enregistrement par fonction

SPF et DMARC sont définis de telle sorte qu’un seul enregistrement approprié peut exister pour un nom donné. Avec SPF, deux enregistrements `v=spf1` entraînent un `permerror`, et le contrôle est donc considéré comme échoué, non comme réussi. Avec DMARC, les destinataires ignorent entièrement le domaine lorsque plusieurs enregistrements commencent par `v=DMARC1` : au lieu d’une politique stricte, aucune ne s’applique.

C’est de loin l’erreur la plus fréquente dans les zones qui ont évolué au fil du temps. Un nouveau prestataire est raccordé, quelqu’un ajoute « son » enregistrement SPF au lieu d’étendre l’existant, et à partir de ce moment le contrôle échoue pour tous les expéditeurs. Il faut donc impérativement vérifier ce qui existe déjà avant chaque nouvel enregistrement :

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT` | Type d’enregistrement interrogé |
| `example.com`, `_dmarc.example.com` | Noms interrogés : le domaine lui-même pour SPF, le nom `_dmarc` pour DMARC |
| `grep -i spf1` | Filtre la ligne SPF ; `-i` ignore la casse |

</details>

Pour DKIM, c’est l’inverse : un enregistrement est prévu par sélecteur, et plusieurs sélecteurs côte à côte sont la norme, car chaque service d’envoi apporte sa propre clé.

### Le suffixe de zone dans les interfaces web

Dans Infoblox, DNS Windows et pratiquement toutes les interfaces d’hébergement, le nom de zone est automatiquement ajouté au nom saisi. Quiconque renseigne le nom pleinement qualifié dans le champ « Nom » obtient un enregistrement deux fois plus long que prévu :

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Dans le fichier de zone, l’équivalent est l’absence du point final. `mail.example.com` sans point final est un nom relatif auquel le nom de zone est ajouté, tandis que `mail.example.com.` avec un point est absolu. Pour les cibles MX et CNAME, ce seul point décide si le domaine est joignable.

### Le copier-coller est la source d’erreur la plus fréquente

Les valeurs des enregistrements de messagerie sont presque toujours copiées depuis un PDF, un ticket, une cellule Excel ou une conversation, rarement saisies au clavier. Cela provoque des dommages invisibles dans le champ de saisie :

- Un double `p=` au début de la clé DKIM, parce que le préfixe a été ajouté deux fois lors de l’assemblage. La valeur `v=DKIM1;k=rsa;p=p=MIIBIjAN...` survient régulièrement en pratique et produit une clé inutilisable.
- Des guillemets typographiques provenant de Word à la place de guillemets droits.
- Des espaces insécables provenant de mises en page PDF, qui ressemblent à des espaces ordinaires.
- Des retours à la ligne au milieu du bloc Base64 lorsque la valeur s’étendait sur plusieurs lignes dans le PDF.

Base64 ne connaît que les caractères A à Z, a à z, 0 à 9, `+`, `/` et `=` comme caractères de remplissage. Tout autre caractère dans la partie `p=` est une erreur. Un court filtre avant la saisie évite les recherches d’erreur ultérieures :

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `'%s' "$KEY"` | Affiche la valeur de la clé telle quelle et sans retour à la ligne ajouté |
| `tr -d 'A-Za-z0-9+/='` | Supprime tous les caractères valides pour Base64 ; seuls les caractères étrangers restent |
| `wc -c` | Compte les caractères restants |

</details>

Si le résultat est différent de `0`, la clé contient des caractères étrangers.

### Réduire le TTL avant les changements

Avant tout changement prévu d’un enregistrement MX, SPF ou DKIM, le TTL doit être réduit pendant quelques heures à une valeur basse, typiquement 300 secondes. Sinon, selon la zone, l’ancienne valeur reste un jour ou plus dans des résolveurs externes, et un retour arrière prend tout autant de temps. Après le changement et une phase d’observation, le TTL est rétabli à sa valeur normale.

## MX

L’enregistrement MX définit quel hôte accepte les e-mails pour le domaine. Deux règles sont régulièrement enfreintes à cet égard.

**La cible doit être un nom d’hôte possédant un enregistrement A ou AAAA.** Ni une adresse IP ni un CNAME ne sont admis. La RFC 2181 indique expressément que la cible d’un enregistrement MX ne doit pas être un alias. En pratique, cela fonctionne malgré tout chez de nombreux destinataires, mais pas chez d’autres, ce qui entraîne des problèmes qui semblent ne concerner que certains expéditeurs.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Le nombre est une préférence, pas une pondération.** La valeur la plus basse est tentée en premier. Un second MX avec une valeur élevée n’est utile que si ce système connaît le même filtrage des destinataires. Les entrées MX de secours sur des systèmes sans vérification des destinataires sont une cible appréciée du spam, car les attaquants visent délibérément l’entrée la plus faible.

Les domaines qui envoient uniquement ou qui n’ont aucun rapport avec la messagerie reçoivent un Null MX selon la RFC 7505. Il signale que le domaine n’accepte aucun e-mail et garantit un refus immédiat et explicite plutôt que des délais d’attente :

```text
example.com.  IN  MX  0 .
```

Le Null MX ne remplace toutefois pas un enregistrement SPF et DMARC. Ne pas recevoir ne signifie pas que personne n’envoie en votre nom. Les sous-domaines parqués sont particulièrement utilisés pour l’usurpation, car personne ne les surveille souvent.

## A, AAAA, PTR et le nom HELO

L’enregistrement PTR de l’adresse IP sortante ne se trouve pas dans votre zone, mais dans la zone `in-addr.arpa` du fournisseur auquel appartient le bloc d’adresses. Il doit donc être commandé auprès du fournisseur et non défini par vous-même. De nombreux grands destinataires exigent que PTR et l’enregistrement direct correspondant concordent, c’est-à-dire que le nom issu du PTR résolve à nouveau vers la même adresse IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `-x 192.0.2.10` | Requête inverse : dig forme lui-même le nom PTR dans la zone `in-addr.arpa` |
| `A mail1.example.com` | Requête directe du nom issu du PTR, afin de vérifier que le parcours revient à la même adresse IP |

</details>

Le nom que votre serveur de messagerie annonce dans HELO ou EHLO devrait être le même et également résoluble. Une passerelle qui se présente comme `localhost.localdomain` ou avec un nom interne est moins bien évaluée par les grands destinataires.

Il convient d’être prudent lors de l’ajout d’un enregistrement AAAA. Dès que le serveur de messagerie est joignable et envoie via IPv6, les mêmes exigences que pour IPv4 s’appliquent, parfois de manière encore plus stricte. Google exige un PTR valide pour les adresses IPv6 émettrices. S’il manque, l’envoi est refusé, alors qu’il fonctionnait parfaitement via IPv4. Un enregistrement AAAA sur le serveur de messagerie n’est donc jamais une simple modification DNS.

## SPF

SPF définit quels systèmes sont autorisés à envoyer au nom du domaine. L’enregistrement est placé comme TXT sur le domaine lui-même.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### La limite de recherches

L’évaluation d’un enregistrement SPF ne peut déclencher plus de dix mécanismes effectuant des requêtes DNS. Sont comptés `include`, `a`, `mx`, `ptr`, `exists` et `redirect`, de manière récursive : chaque `include` apporte les recherches de l’enregistrement inclus. Ne sont pas comptés `ip4`, `ip6` et `all`.

Si la limite est dépassée, le résultat est un `permerror`. Pour DMARC, cela signifie un SPF échoué, que le serveur émetteur soit effectivement autorisé ou non. Le piège est que l’erreur survient souvent sans action de votre part, parce qu’un fournisseur inclus étend son enregistrement. Votre propre enregistrement n’a pas changé, mais la livraison se dégrade malgré tout.

En outre, seules deux « void lookups », c’est-à-dire des requêtes sans résultat, sont autorisées. Un `include` vers un domaine qui n’existe plus entre dans ce décompte. Les références à des prestataires abandonnés doivent donc être supprimées et non conservées par précaution.

### Ce qui n’a pas sa place dans un enregistrement SPF

- **`ptr`** est certes spécifié, mais il est considéré comme obsolète depuis la RFC 7208 et ne doit pas être utilisé. Les systèmes d’évaluation peuvent l’ignorer.
- **`+all`** autorise n’importe quel expéditeur et est donc plus dommageable que l’absence totale d’enregistrement SPF.
- **`?all`** est neutre et donc pratiquement sans valeur pour DMARC.
- **Un enregistrement séparé de type SPF (type 99)** n’est plus nécessaire. Il a été supprimé par la RFC 7208 ; SPF se trouve exclusivement dans TXT.

Le choix entre `~all` (softfail) et `-all` (hardfail) dépend de l’exhaustivité de l’inventaire des voies d’envoi. Tant qu’il existe un doute à ce sujet, `~all` est le bon choix. Ceux qui appliquent déjà DMARC et évaluent les rapports peuvent passer à `-all`.

### Les sous-domaines n’héritent de rien

Un enregistrement SPF sur `example.com` ne s’applique pas à `newsletter.example.com`. Chaque sous-domaine émetteur nécessite son propre enregistrement. Pour tous les autres, une entrée wildcard est recommandée afin d’indiquer clairement qu’aucun envoi ne provient de là :

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attention : un wildcard TXT répond aussi aux requêtes pour des noms comme `_dmarc.sub.example.com`, s’il n’existe pas d’enregistrement explicite à cet endroit. Cela ne pose généralement pas de problème, mais peut compliquer le diagnostic, car chaque requête TXT reçoit une réponse.

### SPF flattening

Les outils qui résolvent toutes les références `include` et les remplacent par les adresses IP correspondantes résolvent la limite de recherches au détriment de la maintenabilité. Si le fournisseur modifie ses adresses, l’envoi échoue sans que personne ne s’en rende compte, car tout semble correct dans votre propre enregistrement. Ceux qui choisissent cette voie ont donc besoin d’un rapprochement automatisé qui vérifie régulièrement la liste par rapport à la source. En travail manuel ponctuel, cette méthode finira tôt ou tard par échouer.

## DKIM

DKIM signe les messages sortants. La clé publique se trouve sous `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Le sélecteur est librement choisi et défini par le système émetteur. Un nom explicite incluant une date facilite beaucoup plus la rotation ultérieure que `s1` et `s2`.

### Délégation par CNAME

Lorsque le service d’envoi le propose, la variante CNAME est préférable à la saisie directe :

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Le fournisseur peut alors faire pivoter sa clé de manière autonome, sans que quelqu’un doive intervenir dans votre zone. Cette rotation reste sinon régulièrement en suspens, car elle exige une coordination entre deux équipes. Un CNAME exclut toutefois tout autre enregistrement portant le même nom ; c’est une règle fondamentale du DNS et non une particularité de DKIM.

### Rotation sans interruption

Lors d’un changement de clé, le nouveau sélecteur est d’abord publié, puis le serveur émetteur bascule dessus, et l’ancien enregistrement n’est supprimé qu’ensuite. Supprimer immédiatement l’ancienne clé invalide les signatures de tous les messages encore en transit ou dans des files d’attente et rend les vérifications a posteriori impossibles. Un délai de quelques jours entre le basculement et la suppression est approprié.

Un enregistrement avec un `p=` vide n’est d’ailleurs pas une entrée défectueuse, mais la manière spécifiée d’indiquer qu’une clé a été retirée.

### Longueur de clé

1024 bits sont considérés comme obsolètes, 2048 bits sont la norme. Des clés RSA plus grandes n’apportent pratiquement aucun avantage supplémentaire et augmentent seulement le risque qu’un système intermédiaire ne traite pas proprement l’enregistrement.

## DMARC

DMARC relie SPF et DKIM à une instruction indiquant ce qui doit se produire lorsqu’un contrôle échoue, et fournit des rapports en retour. L’enregistrement se trouve sous `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Depuis mai 2026, la version révisée définie par la RFC 9989 ainsi que les spécifications de rapports RFC 9990 et RFC 9991 s’applique et remplace la RFC 7489. Trois changements sont importants en pratique :

- **`pct` a été supprimé.** Le déploiement progressif par pourcentage n’existe plus. Il est remplacé par `t=y`, qui identifie le domaine comme étant en phase de test : les rapports continuent, la politique ne doit pas être appliquée.
- **`np` est nouveau.** Il définit la politique pour les sous-domaines inexistants et comble ainsi une lacune volontiers exploitée par les attaquants, car les sous-domaines inventés n’étaient auparavant couverts que par `sp`. Sans indication propre, `np` reprend la valeur de `sp`.
- **La Public Suffix List est remplacée par un `Tree Walk`.** Le domaine organisationnel n’est plus déterminé à partir d’une liste gérée en externe, mais au moyen de requêtes DNS graduelles le long de l’arborescence des noms. Pour les grands espaces de noms à plusieurs niveaux, cela modifie sensiblement l’évaluation.

### L’alignement est le véritable cœur du sujet

DMARC ne réussit pas parce que SPF ou DKIM ont techniquement réussi, mais seulement si au moins l’un des deux correspond également au domaine d’expéditeur visible dans l’en-tête `From`. SPF est alors vérifié par rapport au domaine de l’expéditeur d’enveloppe, qui diffère régulièrement lors de redirections, de services de newsletters et de systèmes de tickets. C’est précisément pourquoi des messages dotés d’un SPF valide échouent parfois au contrôle DMARC.

Avec `adkim=r` et `aspf=r` (relaxed, la valeur par défaut), une correspondance au niveau du domaine organisationnel suffit. `s` exige une égalité exacte, sous-domaine inclus, et échoue en pratique presque toujours sur l’une des voies d’envoi.

### Les adresses de rapport externes nécessitent une autorisation

Si les rapports doivent être envoyés à une adresse située hors de votre propre domaine, par exemple à un service d’analyse DMARC, le domaine destinataire doit l’autoriser. Sans cet enregistrement, de nombreux destinataires n’envoient tout simplement rien et l’analyse reste vide, tandis que tout semble correct dans votre propre enregistrement :

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Cette entrée est créée par l’opérateur de la zone de destination, et non par vous. Avec les services commerciaux, cela se produit automatiquement, mais pas avec une boîte de collecte exploitée par vous-même dans un autre de vos domaines.

### Erreurs de syntaxe typiques

Les noms de tags et les valeurs de politique doivent être écrits en minuscules ; `p=Reject` est invalide. Les tags sont séparés par un point-virgule ; un séparateur manquant rend le reste de la ligne inefficace. Et `p` doit être le premier tag après `v`. Un enregistrement composé uniquement de `v=DMARC1; rua=...` ne contient aucune politique et est incomplet.

### Le déploiement

`p=none` est un état de mesure, pas un objectif. Il ne change rien à la manière dont les destinataires traitent vos e-mails et sert uniquement à identifier, par les rapports, toutes les voies d’envoi légitimes. Ceux qui ne passent pas, quelques mois après l’introduction, de `quarantine` à `reject` ont fait l’effort sans obtenir la protection. L’aspect organisationnel de cette démarche, y compris le document d’aide à la décision, est un sujet à part entière et est décrit dans le blueprint DMARC.

## MTA-STS et TLS-RPT

SMTP chiffre de manière opportuniste : si le système distant propose STARTTLS, la connexion est chiffrée, sinon elle ne l’est pas. Un attaquant en mesure de manipuler le trafic peut supprimer l’annonce STARTTLS et maintenir ainsi la connexion en clair. MTA-STS comble cette lacune pour les domaines destinataires.

MTA-STS se compose de deux parties, dont une seule se trouve dans le DNS :

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La politique proprement dite se présente sous la forme d’un fichier sous `https://mta-sts.example.com/.well-known/mta-sts.txt` et doit être fournie via un certificat valide :

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Les sources d’erreur se situent presque toutes en dehors de la zone :

- **L’`id` doit changer à chaque modification de politique.** C’est le seul indice indiquant aux systèmes émetteurs qu’une nouvelle politique doit être récupérée. Quiconque adapte le fichier sans modifier l’`id` travaille contre des copies mises en cache jusqu’à l’expiration de `max_age`.
- **La liste MX de la politique et les enregistrements MX doivent correspondre.** Un nouveau MX absent de la politique est refusé par les expéditeurs avec `mode: enforce`. Lors des migrations, la politique doit donc être adaptée avant le changement de MX.
- **`mode: testing` en premier.** Dans ce mode, les violations sont seulement signalées, pas appliquées. Le passage à `enforce` a lieu lorsque les rapports sont propres.
- **Un enregistrement CAA peut bloquer l’émission du certificat pour l’hôte de politique**, si une autre autorité de certification y est inscrite que celle utilisée.

TLS-RPT fournit les rapports correspondants et se compose d’un seul enregistrement :

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT est également utile sans MTA-STS, car il rend enfin visibles les échecs de chiffrement de transport.

## DANE

DANE atteint le même objectif que MTA-STS, mais ancre la confiance dans le DNS plutôt que dans l’infrastructure Web PKI. Il nécessite une zone signée de bout en bout avec DNSSEC ; sans DNSSEC, un enregistrement TLSA est inefficace.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Point essentiel en exploitation : à chaque changement de certificat, l’enregistrement TLSA doit correspondre au préalable. La procédure habituelle publie le nouveau hachage en parallèle de l’ancien, change ensuite le certificat, puis supprime l’ancienne entrée. Inverser cet ordre rend le serveur de messagerie inaccessible à tous les expéditeurs vérifiant DANE, parmi lesquels figurent les grands fournisseurs germanophones. En Suisse, DANE est nettement plus rare que MTA-STS, généralement en raison de l’absence de signature DNSSEC de la zone.

## BIMI

BIMI affiche le logo de la marque dans la boîte de réception et constitue le seul mécanisme traité ici qui ne soit pas encore une RFC, mais reste un Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Les conditions sont élevées : une politique DMARC appliquée avec `quarantine` ou `reject`, un logo au format SVG Tiny Portable/Secure et, pour la plupart des fournisseurs, un Verified Mark Certificate payant. BIMI n’est donc pas un mécanisme de sécurité, mais une question de visibilité ; il doit venir à la fin de l’ordre des priorités, non au début.

## Autres enregistrements connexes

**Autodiscover et SRV :** Les environnements Exchange utilisent `autodiscover.example.com` comme CNAME ou un enregistrement SRV `_autodiscover._tcp.example.com`. Les deux concernent la configuration des clients et non le flux de messagerie, mais sont volontiers oubliés lors des migrations et entraînent alors des profils impossibles à configurer.

**CAA :** N’a pas de lien direct avec la messagerie, mais décide quelle autorité de certification peut émettre un certificat pour `mta-sts.example.com` ou le nom du serveur de messagerie.

**Zones split-horizon :** Lorsqu’une zone DNS interne porte le même nom que la zone publique, les enregistrements de messagerie n’existent souvent pas en interne. Les systèmes internes qui effectuent une vérification SPF ou DKIM aboutissent alors à des résultats différents de ceux du monde extérieur. Toute modification des enregistrements de messagerie doit donc s’accompagner de la question de savoir si la zone interne doit être mise à jour.

## Quelques tests rapides

Effectuez délibérément toutes les requêtes auprès d’un résolveur public afin que le cache interne ou une zone split-horizon ne réponde pas :

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `@1.1.1.1` | Envoie la requête à ce résolveur plutôt qu’à celui configuré dans `/etc/resolv.conf` |
| `+short` | Affiche uniquement les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `MX`, `TXT` | Types d’enregistrement interrogés |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Les noms définis sous le domaine pour DMARC, DKIM, MTA-STS et TLS-RPT |

</details>

Contre le serveur faisant autorité, afin d’éviter entièrement le cache :

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `NS example.com` | Détermine les serveurs de noms faisant autorité pour la zone |
| `@ns1.example.com` | Envoie la requête suivante directement à l’un de ces serveurs faisant autorité |
| `+norecurse` | Ne définit pas le bit Recursion Desired ; le serveur répond uniquement à partir de ses propres données de zone, et non d’un cache |

</details>

Sous Windows sans `dig` :

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-type=TXT` | Type d’enregistrement à interroger |
| `_dmarc.example.com` | Nom interrogé |
| `1.1.1.1` | Résolveur à utiliser à la place de celui configuré à l’échelle du système |

</details>

Pour une évaluation complète incluant le décompte des recherches SPF, la recherche de sélecteurs DKIM et le contrôle d’alignement, cette page propose le [contrôle DNS de messagerie](https://rafaelpfister.ch/tools/mail-check), qui vérifie un domaine en une seule fois par rapport à tous les enregistrements décrits ici.

Le test le plus probant reste toutefois un message réel. Envoyez un e-mail vers une boîte chez un grand fournisseur et examinez la ligne `Authentication-Results` dans les en-têtes. Elle indique sur une seule ligne les résultats réels de SPF, DKIM et DMARC, et remplace toute théorie sur le fichier de zone.

## Ordre lors d’une migration

En cas de changement de fournisseur de messagerie, cette séquence a fait ses preuves :

1. Réduire le TTL de tous les enregistrements concernés à 300 secondes, au moins un jour à l’avance.
2. Publier les sélecteurs DKIM du nouveau fournisseur tant que les anciens sont encore présents.
3. Étendre SPF avec le nouveau fournisseur sans supprimer l’ancien, et recalculer à cette occasion la limite de recherches.
4. Pour MTA-STS, adapter la politique aux nouveaux noms MX et augmenter l’`id` avant que les enregistrements MX ne changent.
5. Basculer les MX et observer la livraison.
6. Ne supprimer les anciens includes SPF et sélecteurs DKIM qu’après quelques jours sans incident.
7. Rétablir le TTL.

Le problème le plus fréquent dans cette séquence est de réaliser l’étape 6 trop tôt : les anciennes entrées sont supprimées avec le basculement, et tout ce qui passe encore par l’ancienne voie échoue à l’authentification de l’expéditeur.

## Conclusion

Les enregistrements de messagerie se distinguent de toutes les autres entrées DNS en ce qu’une erreur ne se remarque pas nécessairement. Un enregistrement A erroné génère un ticket en quelques minutes ; un enregistrement SPF en double ou une clé DKIM avec un caractère en trop, en revanche, entraîne un taux de livraison qui baisse lentement pendant des semaines.

Trois règles évitent la plupart de ces cas. Premièrement : avant chaque nouvel enregistrement, vérifiez ce qui existe déjà au lieu d’en ajouter un second à côté. Deuxièmement : après chaque modification, contrôlez auprès d’un résolveur public et comparez la valeur caractère par caractère avec le modèle, pas seulement visuellement. Troisièmement : lors de changements, publiez toujours d’abord le nouveau, basculez ensuite, puis supprimez l’ancien. Ceux qui respectent cet ordre disposent toujours d’une possibilité de retour en arrière pour les enregistrements de messagerie.

## Historique des modifications

- **25 octobre 2026 :** mise à jour du lien de source vers la nouvelle adresse de Microsoft.

## Sources

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): définit notamment la limite de 255 octets d’une seule `character-string` dans les enregistrements TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): indique dans la section 10.3 que la cible d’un enregistrement MX ne doit pas être un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): limite de recherches à dix mécanismes, limite de recherches vides, suppression du type RR SPF et déconseil du mécanisme `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): structure de l’enregistrement de clé sous `_domainkey`, rôle du sélecteur et du `p=` vide.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): spécification DMARC actuelle de mai 2026, remplace la RFC 7489 ; suppression de `pct`, nouveau tag `np`, Tree Walk à la place de la Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): format et acheminement des rapports agrégés, y compris l’autorisation des domaines destinataires externes.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): identification des domaines qui n’acceptent aucun e-mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): enregistrement DNS, fichier de politique, rôle de l’`id` et des modes `testing` et `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): structure de l’enregistrement `_smtp._tls` et des rapports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): enregistrements TLSA pour SMTP et condition préalable d’une zone signée avec DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): état actuel de la spécification BIMI, toujours pas une RFC.

12.  [Google: Directives pour les expéditeurs d’e-mails](https://support.google.com/a/answer/81126): exigences applicables aux expéditeurs, notamment l’obligation de PTR pour les adresses IPv6 émettrices et les exigences pour les expéditeurs de masse en vigueur depuis février 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399730): exigences pour les expéditeurs à partir de 5000 messages par jour, valables depuis mai 2025.
