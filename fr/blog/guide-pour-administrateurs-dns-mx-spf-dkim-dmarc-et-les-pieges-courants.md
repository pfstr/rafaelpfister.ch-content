---
title: "Guide pour administrateurs DNS : MX, SPF, DKIM, DMARC et les pièges courants"
navTitle: "Enregistrements DNS d’e-mail"
description: "Les personnes qui gèrent une zone reçoivent généralement les enregistrements de messagerie déjà prêts et n’ont plus qu’à les publier. Ce qui échoue régulièrement : la limite de 255 octets pour DKIM, les enregistrements SPF en double, la limite de recherches, un MX sur un CNAME, le suffixe de zone ajouté automatiquement et des politiques que plus personne n’applique."
date: "2026-08-04"
kategorie: "SMTP et flux de messagerie"
timeToRead: "15 min de lecture"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
produkte:
  - "uebergreifend"
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
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
url: https://rafaelpfister.ch/fr/blog/guide-pour-administrateurs-dns-mx-spf-dkim-dmarc-et-les-pieges-courants
translationSourceHash: dc806bed491a47ecc1118249566d9303b0201f4bdb5153a966385a7c9373b31f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-07T05:12:57.490Z
translationReview: automatic
---

# Guide pour administrateurs DNS : MX, SPF, DKIM, DMARC et les pièges courants

Les personnes qui gèrent une zone DNS reçoivent rarement les enregistrements de messagerie rédigés par elles-mêmes. L’équipe de messagerie, un fournisseur ou un prestataire marketing envoie une ligne en précisant qu’elle doit « simplement être publiée ». C’est précisément là que surviennent la plupart des erreurs, car les enregistrements de messagerie sont le type d’enregistrement pour lequel une faute de frappe peut avoir deux conséquences totalement différentes. Soit la distribution échoue immédiatement et quelqu’un se manifeste dans les minutes qui suivent, soit elle continue sans changement et seule l’authentification de l’expéditeur échoue silencieusement. Le second cas passe régulièrement inaperçu pendant des mois, jusqu’à ce qu’un grand destinataire place le domaine en quarantaine.

Depuis que Google et Yahoo ont renforcé leurs exigences pour les expéditeurs de masse en février 2024 et que Microsoft a suivi en mai 2025, la tolérance envers les domaines partiellement configurés est devenue faible. SPF, DKIM et un enregistrement DMARC ne sont plus facultatifs pour les expéditeurs à partir d’un certain volume, mais une condition de distribution.

Tous les exemples de cet article utilisent `example.com` et des sélecteurs génériques. Les valeurs présentées sont abrégées afin de rester lisibles.

## Règles valables pour chaque enregistrement de messagerie

### La limite de 255 octets des enregistrements TXT

Selon la RFC 1035, un enregistrement TXT se compose d’une ou plusieurs `character-strings`, et chacune de ces chaînes peut contenir au maximum 255 octets. L’enregistrement dans son ensemble peut être plus long, mais il doit alors être divisé en plusieurs chaînes. Les systèmes d’évaluation assemblent à nouveau ces parties sans séparateur.

Cela devient concrètement pertinent à un endroit précis : les clés DKIM de 2048 bits. Leur valeur Base64 compte environ 400 caractères et ne tient pas dans une seule chaîne.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La plupart des systèmes de gestion DNS effectuent eux-mêmes cette répartition lorsque la valeur est saisie via le champ de saisie habituel. En revanche, si vous ajoutez manuellement des guillemets, vous devez respecter exactement la limite. Une valeur divisée avec un espace à la jonction produit une clé qui existe syntaxiquement mais ne correspond plus cryptographiquement.

Le contrôle ultérieur est important, car une clé mal assemblée paraît parfaitement normale dans l’interface :

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### Un enregistrement par usage

SPF et DMARC sont définis de telle sorte qu’un seul enregistrement correspondant peut exister pour un nom donné. Pour SPF, deux enregistrements `v=spf1` provoquent un `permerror`, et le contrôle est alors considéré comme échoué, non comme réussi. Pour DMARC, les destinataires ignorent entièrement le domaine lorsque plusieurs enregistrements commencent par `v=DMARC1` : au lieu d’une politique stricte, aucune ne s’applique.

C’est de loin l’erreur la plus fréquente dans les zones ayant évolué au fil du temps. Un nouveau prestataire est raccordé, quelqu’un ajoute « son » enregistrement SPF au lieu d’étendre celui qui existe, et dès lors le contrôle échoue pour tous les expéditeurs. Avant tout nouvel enregistrement, vérifiez donc impérativement ce qui existe déjà :

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

Pour DKIM, c’est l’inverse : un enregistrement est prévu par sélecteur, et plusieurs sélecteurs côte à côte constituent le cas normal, car chaque service d’envoi apporte sa propre clé.

### Le suffixe de zone dans les interfaces web

Dans Infoblox, DNS Windows et pratiquement toutes les interfaces d’hébergement, le nom de zone est automatiquement ajouté au nom saisi. Si vous entrez le nom pleinement qualifié dans le champ « Nom », vous obtenez un enregistrement deux fois plus long que prévu :

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Dans le fichier de zone, l’équivalent est l’absence du point final. `mail.example.com` sans point à la fin est un nom relatif auquel le nom de zone est ajouté, tandis que `mail.example.com.` avec un point est absolu. Pour les cibles MX et CNAME, ce seul point détermine si le domaine est joignable.

### Le copier-coller est la source d’erreurs la plus fréquente

Les valeurs des enregistrements de messagerie sont presque jamais saisies au clavier, mais copiées depuis un PDF, un ticket, une cellule Excel ou une discussion. Cela peut causer des dommages invisibles dans le champ de saisie :

- Un `p=` double au début de la clé DKIM, parce que le préfixe a été défini deux fois lors de l’assemblage. La valeur `v=DKIM1;k=rsa;p=p=MIIBIjAN...` est un classique réel et produit une clé inutilisable.
- Des guillemets typographiques issus de Word au lieu de guillemets droits.
- Des espaces insécables provenant de mises en page PDF, qui ressemblent à des espaces normaux.
- Des retours à la ligne au milieu du bloc Base64 lorsque la valeur s’étendait sur plusieurs lignes dans le PDF.

Base64 ne connaît que les caractères A à Z, a à z, 0 à 9, `+`, `/` et `=` comme caractères de remplissage. Tout autre caractère dans la partie `p=` constitue une erreur. Un filtre court avant la saisie évite les recherches de panne ultérieures :

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

Si le résultat est autre chose que `0`, la clé contient des caractères étrangers.

### Réduire le TTL avant les changements

Avant tout changement planifié d’un enregistrement MX, SPF ou DKIM, le TTL doit être abaissé pendant quelques heures à une valeur faible, typiquement 300 secondes. Sinon, selon la zone, l’ancienne valeur reste encore un jour ou davantage dans les résolveurs externes, et un retour en arrière prend tout autant de temps. Après le changement et une phase d’observation, le TTL est rétabli à sa valeur habituelle.

## MX

L’enregistrement MX définit quel hôte accepte les e-mails pour le domaine. Deux règles sont régulièrement enfreintes à ce sujet.

**La cible doit être un nom d’hôte avec un enregistrement A ou AAAA.** Ni une adresse IP ni un CNAME ne sont autorisés. La RFC 2181 indique expressément que la cible d’un enregistrement MX ne doit pas être un alias. En pratique, cela fonctionne malgré tout chez de nombreux destinataires, mais pas chez d’autres, ce qui conduit à des problèmes semblant ne toucher que certains expéditeurs.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Le nombre est une préférence, pas une pondération.** La valeur la plus basse est tentée en premier. Un second MX avec une valeur élevée n’a de sens que si ce système connaît le même filtrage des destinataires. Les entrées MX de secours vers des systèmes sans contrôle des destinataires sont une cible appréciée du spam, car les attaquants visent délibérément l’entrée la plus faible.

Les domaines qui envoient uniquement ou n’ont aucun lien avec la messagerie reçoivent un Null MX conformément à la RFC 7505. Il signale que le domaine n’accepte pas d’e-mails et garantit un refus immédiat et clair plutôt que des délais d’attente :

```text
example.com.  IN  MX  0 .
```

Le Null MX ne remplace toutefois pas un enregistrement SPF et DMARC. Ne pas recevoir ne signifie pas que personne n’envoie en votre nom. Les sous-domaines secondaires parqués sont particulièrement utilisés pour l’usurpation, car ils sont rarement surveillés.

## A, AAAA, PTR et le nom HELO

L’enregistrement PTR de l’adresse IP sortante ne se trouve pas dans votre zone, mais dans la zone `in-addr.arpa` du fournisseur auquel appartient le bloc d’adresses. Il doit donc être commandé auprès du fournisseur et non configuré par vous-même. De nombreux grands destinataires exigent que le PTR et l’enregistrement direct correspondant concordent, c’est-à-dire que le nom du PTR se résolve à nouveau vers la même adresse IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

Le nom que votre serveur de messagerie annonce dans HELO ou EHLO devrait être identique et également résoluble. Une passerelle qui se présente comme `localhost.localdomain` ou avec un nom interne est moins bien évaluée par les grands destinataires.

La prudence est requise lors de l’ajout d’un enregistrement AAAA. Dès que le serveur de messagerie est joignable et émet via IPv6, les mêmes exigences s’appliquent que pour IPv4, parfois même de manière plus stricte. Google exige un PTR valide pour les adresses IPv6 émettrices. S’il manque, l’envoi est refusé, alors qu’il fonctionnait parfaitement via IPv4. Un enregistrement AAAA sur le serveur de messagerie n’est donc jamais une simple modification DNS.

## SPF

SPF définit quels systèmes sont autorisés à envoyer au nom du domaine. L’enregistrement est placé en TXT à la racine du domaine.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### La limite de recherches

L’évaluation d’un enregistrement SPF peut déclencher au maximum dix mécanismes effectuant des requêtes DNS. Sont comptés `include`, `a`, `mx`, `ptr`, `exists` et `redirect`, de manière récursive : chaque `include` apporte les recherches de l’enregistrement inclus. Ne sont pas comptés `ip4`, `ip6` et `all`.

Si la limite est dépassée, le résultat est un `permerror`. Pour DMARC, cela signifie un SPF échoué, indépendamment du fait que le serveur émetteur aurait en réalité été autorisé. Le piège est que l’erreur survient souvent sans intervention de votre part, parce qu’un fournisseur inclus étend son enregistrement. Votre propre enregistrement n’a pas changé, mais la distribution se dégrade tout de même.

En outre, seuls deux « void lookups », c’est-à-dire des requêtes sans résultat, sont autorisés. Un `include` vers un domaine qui n’existe plus entre dans ce calcul. Les références à des prestataires abandonnés doivent donc être supprimées et non conservées par précaution.

### Ce qui n’a pas sa place dans un enregistrement SPF

- **`ptr`** est certes spécifié, mais est considéré comme obsolète depuis la RFC 7208 et ne doit pas être utilisé. Les systèmes d’évaluation peuvent l’ignorer.
- **`+all`** autorise n’importe quel expéditeur et est donc plus nuisible que l’absence totale d’enregistrement SPF.
- **`?all`** est neutre et donc pratiquement inutile pour DMARC.
- **Un enregistrement séparé de type SPF (type 99)** n’est plus nécessaire. Il est abandonné depuis la RFC 7208 ; SPF se trouve exclusivement dans TXT.

Le choix entre `~all` (softfail) et `-all` (hardfail) dépend de l’exhaustivité des chemins d’envoi recensés. Tant qu’il existe un doute à ce sujet, `~all` est le bon choix. Les personnes qui appliquent déjà DMARC et évaluent les rapports peuvent passer à `-all`.

### Les sous-domaines n’héritent de rien

Un enregistrement SPF sur `example.com` ne s’applique pas à `newsletter.example.com`. Chaque sous-domaine émetteur nécessite son propre enregistrement. Pour tous les autres, une entrée générique indiquant clairement qu’aucun envoi ne provient de là est recommandée :

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attention : un wildcard TXT répond aussi aux requêtes pour des noms tels que `_dmarc.sub.example.com`, à moins qu’un enregistrement explicite n’y existe. C’est généralement sans problème, mais peut compliquer le dépannage, car toute requête TXT reçoit une réponse.

### SPF Flattening

Les outils qui résolvent toutes les références `include` et les remplacent par les adresses IP sous-jacentes résolvent la limite de recherches au prix de la maintenabilité. Si le fournisseur modifie ses adresses, l’envoi échoue, sans que personne ne le remarque, car tout semble correct dans votre propre enregistrement. Ceux qui choisissent cette approche ont donc besoin d’un contrôle automatisé qui vérifie régulièrement la liste par rapport à la source. En tant que travail manuel ponctuel, cette méthode échouera tôt ou tard.

## DKIM

DKIM signe les messages sortants. La clé publique se trouve sous `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Le sélecteur est librement choisi et défini par le système d’envoi. Un nom explicite avec une date facilite bien davantage la rotation ultérieure que `s1` et `s2`.

### Délégation par CNAME

Lorsque le service d’envoi le propose, la variante CNAME est à préférer à la saisie directe :

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Le fournisseur peut alors faire tourner sa clé de manière autonome, sans qu’une personne doive intervenir dans votre zone. Cette rotation est sinon régulièrement oubliée, car elle nécessite une coordination entre deux équipes. Un CNAME exclut toutefois tout autre enregistrement au même nom ; il s’agit d’une règle fondamentale du DNS, et non d’une particularité de DKIM.

### Rotation sans interruption

Lors d’un changement de clé, le nouveau sélecteur est d’abord publié, puis le serveur émetteur est basculé vers celui-ci, et ce n’est qu’ensuite que l’ancien enregistrement est supprimé. Supprimer immédiatement l’ancienne clé invalide les signatures de tous les messages encore en transit ou dans des files d’attente et rend les vérifications ultérieures impossibles. Un délai de quelques jours entre le basculement et la suppression est approprié.

Un enregistrement avec un `p=` vide n’est d’ailleurs pas une entrée défectueuse, mais la manière spécifiée d’indiquer qu’une clé est retirée.

### Longueur de clé

1024 bits sont considérés comme obsolètes, 2048 bits sont la norme. Des clés RSA plus grandes n’apportent pratiquement aucun bénéfice supplémentaire et augmentent seulement la probabilité qu’un système intermédiaire ne traite pas correctement l’enregistrement.

## DMARC

DMARC relie SPF et DKIM à une instruction indiquant ce qui doit se produire lorsqu’un contrôle échoue, et renvoie des rapports. L’enregistrement se trouve sous `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Depuis mai 2026, la version révisée avec la RFC 9989 ainsi que les spécifications de rapports RFC 9990 et RFC 9991 s’applique et remplace la RFC 7489. Trois changements sont importants en pratique :

- **`pct` a été supprimé.** L’introduction progressive via un pourcentage n’existe plus. Il est remplacé par `t=y`, qui désigne le domaine comme étant en phase de test : les rapports continuent d’être envoyés, mais la politique ne doit pas être appliquée.
- **`np` est nouveau.** Il définit la politique pour les sous-domaines inexistants et comble ainsi une lacune que les attaquants exploitent volontiers, car les sous-domaines inventés n’étaient jusque-là couverts que par `sp`. Sans indication propre, `np` suit la valeur de `sp`.
- **La Public Suffix List est remplacée par une `Tree Walk`.** Le domaine organisationnel n’est plus déterminé à partir d’une liste maintenue de manière externe, mais par des requêtes DNS graduelles le long de l’arborescence des noms. Cela modifie sensiblement l’évaluation pour les grands espaces de noms à plusieurs niveaux.

### L’alignement est le véritable cœur du sujet

DMARC ne réussit pas parce que SPF ou DKIM ont techniquement réussi, mais seulement si au moins l’un des deux correspond en plus au domaine de l’expéditeur visible dans l’en-tête `From`. SPF est alors vérifié par rapport au domaine de l’expéditeur d’enveloppe, qui diffère régulièrement lors de redirections, de services de newsletters et de systèmes de tickets. C’est précisément pourquoi des messages avec un SPF valide ne passent parfois pas le contrôle DMARC.

Avec `adkim=r` et `aspf=r` (relaxed, la valeur par défaut), une correspondance au niveau du domaine organisationnel suffit. `s` exige une égalité exacte, sous-domaine inclus, et échoue en pratique presque toujours sur l’un des chemins d’envoi.

### Les adresses de rapport externes nécessitent une autorisation

Si les rapports doivent être envoyés à une adresse en dehors de votre propre domaine, par exemple à un service d’analyse DMARC, le domaine destinataire doit l’autoriser. Sans cet enregistrement, de nombreux destinataires n’envoient tout simplement rien, et l’analyse reste vide alors que tout semble correct dans votre propre enregistrement :

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Cette entrée est créée par l’exploitant de la zone cible, et non par vous. Pour les services commerciaux, cela se fait automatiquement, mais pas pour une boîte de collecte exploitée par vous-même dans un autre domaine qui vous appartient.

### Erreurs de syntaxe typiques

Les noms de tags et les valeurs de politique doivent être écrits en minuscules ; `p=Reject` est invalide. Les tags sont séparés par un point-virgule ; un séparateur manquant rend le reste de la ligne inopérant. Et `p` doit être le premier tag après `v`. Un enregistrement composé uniquement de `v=DMARC1; rua=...` ne contient aucune politique et est incomplet.

### Le déploiement

`p=none` est un état de mesure, pas un objectif. Il ne modifie pas le traitement de vos e-mails par les destinataires et sert uniquement à identifier tous les chemins d’envoi légitimes via les rapports. Toute personne qui ne passe pas, dans les quelques mois suivant l’introduction, de `quarantine` à `reject` a fourni l’effort sans obtenir la protection. L’aspect organisationnel de cette démarche, y compris le document de décision, constitue un sujet distinct et est décrit dans le blueprint DMARC.

## MTA-STS et TLS-RPT

SMTP chiffre de manière opportuniste : si le système distant propose STARTTLS, la connexion est chiffrée, sinon non. Un attaquant en mesure de manipuler le trafic peut supprimer l’annonce STARTTLS et maintenir ainsi la connexion en clair. MTA-STS comble cette lacune pour les domaines destinataires.

MTA-STS se compose de deux parties, dont une seule se trouve dans le DNS :

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La politique proprement dite est un fichier accessible sous `https://mta-sts.example.com/.well-known/mta-sts.txt` et doit être servie via un certificat valide :

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Les pièges se situent presque tous en dehors de la zone :

- **L’`id` doit changer à chaque modification de politique.** C’est le seul indice indiquant aux systèmes émetteurs qu’une nouvelle politique doit être récupérée. Modifier le fichier sans changer l’`id` revient à travailler contre des copies mises en cache jusqu’à l’expiration de `max_age`.
- **La liste MX de la politique et les enregistrements MX doivent correspondre.** Un nouveau MX absent de la politique est rejeté par les émetteurs avec `mode: enforce`. Lors des migrations, la politique doit donc être adaptée avant le changement des MX.
- **`mode: testing` d’abord.** Dans ce mode, les violations sont uniquement signalées, sans être appliquées. Le passage à `enforce` intervient lorsque les rapports sont corrects.
- **Un enregistrement CAA peut bloquer l’émission du certificat pour l’hôte de politique**, si une autre autorité de certification y est indiquée que celle utilisée.

TLS-RPT fournit les rapports associés et consiste en un seul enregistrement :

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT est également utile sans MTA-STS, car il rend enfin visibles les échecs de chiffrement du transport.

## DANE

DANE atteint le même objectif que MTA-STS, mais ancre la confiance dans le DNS plutôt que dans la PKI Web. Il nécessite une zone entièrement signée avec DNSSEC ; sans DNSSEC, un enregistrement TLSA est sans effet.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Point décisif en exploitation : à chaque changement de certificat, l’enregistrement TLSA doit correspondre auparavant. La procédure habituelle publie le nouveau hash en parallèle de l’ancien, change ensuite le certificat, puis supprime l’ancienne entrée. Inverser cet ordre rend le serveur de messagerie inaccessible à tous les émetteurs vérifiant DANE, parmi lesquels figurent les grands fournisseurs germanophones. En Suisse, DANE est nettement moins fréquent que MTA-STS, principalement en raison de l’absence de signature DNSSEC de la zone.

## BIMI

BIMI affiche le logo de la marque dans la boîte de réception et est le seul mécanisme traité ici qui ne soit pas encore une RFC, mais qui reste présenté comme Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Les conditions sont élevées : une politique DMARC appliquée avec `quarantine` ou `reject`, un logo au format SVG Tiny Portable/Secure et, pour la plupart des fournisseurs, un Verified Mark Certificate payant. BIMI n’est donc pas un mécanisme de sécurité, mais un sujet de visibilité, et doit venir à la fin de l’ordre des priorités, non au début.

## Autres enregistrements connexes

**Autodiscover et SRV :** Les environnements Exchange utilisent `autodiscover.example.com` comme CNAME ou un enregistrement SRV `_autodiscover._tcp.example.com`. Les deux concernent la configuration des clients et non le flux de messagerie, mais sont volontiers oubliés lors des migrations et entraînent alors des profils impossibles à configurer.

**CAA :** N’a pas de lien direct avec la messagerie, mais détermine quelle autorité de certification peut émettre un certificat pour `mta-sts.example.com` ou le nom du serveur de messagerie.

**Zones split-horizon :** Lorsqu’une zone DNS interne porte le même nom que la zone publique, les enregistrements de messagerie n’existent souvent pas en interne. Les systèmes internes qui effectuent un contrôle SPF ou DKIM obtiennent alors des résultats différents de ceux du monde extérieur. Toute modification des enregistrements de messagerie doit donc inclure la question de savoir si la zone interne doit être mise à jour.

## Quelques tests rapides

Effectuez volontairement toutes les requêtes auprès d’un résolveur public afin que le cache interne ou une zone split-horizon ne réponde pas :

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Contre le serveur faisant autorité, afin de contourner entièrement le cache :

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

Sous Windows sans `dig` :

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

Pour l’évaluation complète, y compris le comptage des recherches SPF, la recherche de sélecteurs DKIM et le contrôle d’alignement, cette page propose le [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), qui vérifie un domaine en une seule fois par rapport à tous les enregistrements décrits ici.

Le test le plus probant reste toutefois un véritable message. Envoyez un e-mail vers une boîte chez un grand fournisseur et examinez la ligne `Authentication-Results` dans les en-têtes. Elle indique en une ligne les résultats réels de SPF, DKIM et DMARC, et remplace toute théorie sur le fichier de zone.

## Ordre lors d’une migration

Lors d’un changement de fournisseur de messagerie, cette séquence a fait ses preuves :

1. Abaisser le TTL de tous les enregistrements concernés à 300 secondes, au moins un jour à l’avance.
2. Publier les sélecteurs DKIM du nouveau fournisseur tant que les anciens sont encore présents.
3. Étendre SPF avec le nouveau fournisseur, sans supprimer l’ancien, et recalculer la limite de recherches.
4. Pour MTA-STS, adapter la politique aux nouveaux noms MX et augmenter l’`id` avant le changement des enregistrements MX.
5. Basculer les MX et observer la distribution.
6. Supprimer les anciens includes SPF et sélecteurs DKIM seulement après quelques jours sans incident.
7. Rétablir le TTL.

Le problème le plus fréquent dans cette séquence est une étape 6 effectuée trop tôt : les anciennes entrées sont supprimées en même temps que le basculement, et tout ce qui passe encore par l’ancien chemin échoue lors de l’authentification de l’expéditeur.

## Conclusion

Les enregistrements de messagerie se distinguent de toutes les autres entrées DNS en ce qu’une erreur ne se remarque pas nécessairement. Un enregistrement A erroné génère un ticket en quelques minutes ; un enregistrement SPF en double ou une clé DKIM avec un caractère de trop, en revanche, entraîne un taux de distribution qui diminue lentement pendant des semaines.

Trois règles évitent la plupart de ces cas. Premièrement : avant tout nouvel enregistrement, vérifier ce qui existe déjà plutôt que d’en ajouter un deuxième à côté. Deuxièmement : après chaque modification, contrôler auprès d’un résolveur public et comparer la valeur caractère par caractère avec le modèle, pas seulement visuellement. Troisièmement : lors des changements, toujours publier d’abord le nouveau, basculer ensuite, puis supprimer l’ancien. En respectant cet ordre, vous disposez toujours d’une solution de repli pour les enregistrements de messagerie.

## Sources

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Définit notamment la limite de 255 octets d’une seule `character-string` dans les enregistrements TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Indique dans la section 10.3 que la cible d’un enregistrement MX ne doit pas être un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Limite de recherches de dix mécanismes, limite des void lookups, abandon du type RR SPF et recommandation de ne pas utiliser le mécanisme `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Structure de l’enregistrement de clé sous `_domainkey`, signification du sélecteur et du `p=` vide.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Spécification DMARC actuelle de mai 2026, remplace la RFC 7489 ; suppression de `pct`, nouveau tag `np`, Tree Walk au lieu de Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format et distribution des rapports agrégés, y compris l’autorisation des domaines destinataires externes.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identification des domaines qui n’acceptent aucun e-mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Enregistrement DNS, fichier de politique, signification de l’`id` et des modes `testing` et `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Structure de l’enregistrement `_smtp._tls` et des rapports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Enregistrements TLSA pour SMTP et condition préalable d’une zone signée DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): État actuel de la spécification BIMI, toujours pas une RFC.

12.  [Google: Règles applicables aux expéditeurs d’e-mails](https://support.google.com/a/answer/81126): Exigences pour les expéditeurs, notamment l’obligation de PTR pour les adresses IPv6 émettrices et les exigences pour les expéditeurs de masse applicables depuis février 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Exigences pour les expéditeurs à partir de 5000 messages par jour, applicables depuis mai 2025.
