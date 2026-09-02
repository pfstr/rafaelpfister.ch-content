---
title: "Guide pour les administrateurs DNS : MX, SPF, DKIM, DMARC et les sources d’erreurs habituelles"
navTitle: "Enregistrements DNS de messagerie"
description: "Les personnes qui gèrent une zone reçoivent généralement les enregistrements de messagerie prêts à l’emploi et n’ont plus qu’à les publier. Ce qui échoue régulièrement : la limite de 255 octets pour DKIM, les enregistrements SPF en double, la limite de recherches, un MX sur un CNAME, le suffixe de zone ajouté automatiquement et les politiques que plus personne n’applique."
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
translationSourceHash: 63c8a888f2ebd4548bd4222c4273896228649bf02f0406082ec337194af65280
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:19:19.735Z
translationReview: required
url: https://rafaelpfister.ch/fr/blog/guide-pour-administrateurs-dns-mx-spf-dkim-dmarc-et-les-pieges-courants
---

# Guide pour les administrateurs DNS : MX, SPF, DKIM, DMARC et les sources d’erreurs habituelles

Les personnes qui gèrent une zone DNS reçoivent rarement les enregistrements de messagerie rédigés par elles-mêmes. L’équipe messagerie, un fournisseur ou un prestataire marketing envoie une ligne en précisant qu’elle doit « seulement être publiée ». C’est précisément là que surviennent la plupart des erreurs, car les enregistrements de messagerie sont le type d’enregistrement où une faute de frappe peut avoir deux conséquences totalement différentes. Soit la remise échoue immédiatement et quelqu’un se manifeste dans les minutes qui suivent, soit elle continue sans changement et seule la vérification de l’expéditeur échoue silencieusement. Le deuxième cas passe régulièrement inaperçu pendant des mois, jusqu’à ce qu’un grand destinataire place le domaine en quarantaine.

Depuis que Google et Yahoo ont renforcé leurs exigences pour les expéditeurs de masse en février 2024 et que Microsoft leur a emboîté le pas en mai 2025, la tolérance envers les domaines configurés à moitié est devenue faible. SPF, DKIM et un enregistrement DMARC ne sont plus un simple plus pour les expéditeurs dépassant un certain volume, mais une condition de remise.

Tous les exemples de cet article utilisent `example.com` et des sélecteurs génériques. Les valeurs affichées sont raccourcies afin de rester lisibles.

## Règles valables pour chaque enregistrement de messagerie

### La limite de 255 octets des enregistrements TXT

Selon la RFC 1035, un enregistrement TXT est composé d’une ou plusieurs `character-strings`, et chacune de ces chaînes de caractères contient au maximum 255 octets. L’enregistrement dans son ensemble peut être plus long, mais il doit alors être divisé en plusieurs chaînes de caractères. Les systèmes qui l’évaluent réassemblent ces parties sans séparateur.

Cela devient concrètement pertinent à un endroit précis : pour les clés DKIM de 2048 bits. Leur valeur Base64 fait environ 400 caractères et ne tient pas dans une seule chaîne de caractères.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La plupart des systèmes de gestion DNS effectuent eux-mêmes cette répartition lorsque la valeur est saisie dans le champ de saisie standard. En revanche, si l’on ajoute manuellement les guillemets, il faut respecter exactement la limite. Une valeur coupée avec un espace à la jonction produit une clé qui existe syntaxiquement mais qui ne correspond plus cryptographiquement.

Le contrôle ultérieur est important, car une clé mal assemblée paraît parfaitement normale dans l’interface graphique :

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT selector1._domainkey.example.com` | Type d’enregistrement et nom de l’enregistrement de clé DKIM |
| `tr -d '" '` | Supprime les guillemets et les espaces, et réassemble donc les chaînes partielles telles qu’un vérificateur les lit |
| `wc -c` | Compte les caractères de la valeur assemblée ; la longueur doit correspondre au modèle |

</details>

### Un enregistrement par objectif

SPF et DMARC sont définis de manière à ce qu’un seul enregistrement correspondant soit autorisé pour un nom donné. Pour SPF, deux enregistrements `v=spf1` entraînent un `permerror`, et la vérification est alors considérée comme échouée, et non réussie. Pour DMARC, les destinataires ignorent complètement le domaine lorsque plusieurs enregistrements commencent par `v=DMARC1` : au lieu d’une politique stricte, aucune politique ne s’applique alors.

C’est de loin l’erreur la plus fréquente dans les zones qui ont évolué au fil du temps. Un nouveau prestataire est raccordé, quelqu’un ajoute « son » enregistrement SPF au lieu d’étendre celui qui existe, et dès cet instant la vérification échoue pour tous les expéditeurs. Avant tout nouvel enregistrement, il faut donc impérativement vérifier ce qui existe déjà :

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `TXT` | Type d’enregistrement interrogé |
| `example.com`, `_dmarc.example.com` | Noms interrogés : le domaine lui-même pour SPF, le nom `_dmarc` pour DMARC |
| `grep -i spf1` | Filtre la ligne SPF ; `-i` ignore la casse |

</details>

Pour DKIM, c’est l’inverse : un enregistrement est prévu par sélecteur, et plusieurs sélecteurs côte à côte constituent la norme, car chaque service d’envoi apporte sa propre clé.

### Le suffixe de zone dans les interfaces Web

Dans Infoblox, DNS Windows et pratiquement toutes les interfaces d’hébergement, le nom de zone est automatiquement ajouté au nom saisi. Si l’on entre le nom pleinement qualifié dans le champ « Nom », on obtient un enregistrement deux fois plus long que prévu :

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Dans le fichier de zone, l’équivalent est le point final manquant. `mail.example.com` sans point à la fin est un nom relatif et est complété par le nom de zone, tandis que `mail.example.com.` avec un point est absolu. Pour les cibles MX et CNAME, ce seul point détermine si le domaine est joignable.

### Le copier-coller est la source d’erreurs la plus fréquente

Les valeurs des enregistrements de messagerie ne sont presque jamais saisies au clavier, mais copiées depuis un PDF, un ticket, une cellule Excel ou une discussion. Cela crée des problèmes qui restent invisibles dans le champ de saisie :

- Un `p=` en double au début de la clé DKIM, parce que le préfixe a été défini deux fois lors de l’assemblage. La valeur `v=DKIM1;k=rsa;p=p=MIIBIjAN...` survient régulièrement en pratique et produit une clé inutilisable.
- Des guillemets typographiques issus de Word au lieu de guillemets droits.
- Des espaces insécables provenant de mises en page PDF, qui ressemblent à des espaces normaux.
- Des retours à la ligne au milieu du bloc Base64 lorsque la valeur s’étendait sur plusieurs lignes dans le PDF.

Base64 ne connaît que les caractères A à Z, a à z, 0 à 9, `+`, `/` et `=` comme caractères de remplissage. Tout autre caractère dans la partie `p=` est une erreur. Un filtre rapide avant la saisie évite les recherches d’erreurs ultérieures :

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `'%s' "$KEY"` | Affiche la valeur de la clé sans modification et sans retour à la ligne ajouté |
| `tr -d 'A-Za-z0-9+/='` | Supprime tous les caractères valides pour Base64 ; seuls les caractères étrangers restent |
| `wc -c` | Compte les caractères restants |

</details>

Si le résultat est autre chose que `0`, la clé contient des caractères étrangers.

### Réduire le TTL avant les changements

Avant chaque changement planifié d’un enregistrement MX, SPF ou DKIM, le TTL doit être abaissé pendant quelques heures à une valeur faible, généralement 300 secondes. Sinon, selon la zone, l’ancienne valeur reste un jour ou plus dans les résolveurs tiers, et un retour en arrière prend tout aussi longtemps. Après le changement et une phase d’observation, le TTL est remis à sa valeur habituelle.

## MX

L’enregistrement MX définit quel hôte accepte les e-mails pour le domaine. Deux règles sont régulièrement enfreintes à ce sujet.

**La cible doit être un nom d’hôte disposant d’un enregistrement A ou AAAA.** Ni une adresse IP ni un CNAME ne sont autorisés. La RFC 2181 précise explicitement que la cible d’un enregistrement MX ne peut pas être un alias. En pratique, cela fonctionne malgré tout avec de nombreux destinataires, mais pas avec d’autres, ce qui entraîne des problèmes qui semblent ne concerner que certains expéditeurs.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Le nombre est une préférence, pas une pondération.** La valeur la plus basse est tentée en premier. Un second MX avec une valeur élevée n’est utile que si ce système connaît le même filtrage des destinataires. Les entrées MX de secours pointant vers des systèmes sans contrôle des destinataires constituent une cible privilégiée pour le spam, car les attaquants visent délibérément l’entrée la plus faible.

Les domaines qui envoient exclusivement ou n’ont aucun lien avec la messagerie reçoivent un Null MX selon la RFC 7505. Il signale que le domaine n’accepte pas d’e-mails et assure un rejet immédiat et clair plutôt que des délais d’expiration :

```text
example.com.  IN  MX  0 .
```

Le Null MX ne remplace toutefois pas un enregistrement SPF et DMARC. Ne pas recevoir ne signifie pas que personne n’envoie en votre nom. Les sous-domaines parqués, en particulier, sont utilisés pour l’usurpation d’identité, car personne ne les surveille rarement.

## A, AAAA, PTR et le nom HELO

L’enregistrement PTR de l’adresse IP sortante ne se trouve pas dans votre zone, mais dans la zone `in-addr.arpa` du fournisseur auquel appartient le bloc d’adresses. Il doit donc être commandé auprès du fournisseur et ne peut pas être défini soi-même. De nombreux grands destinataires exigent que le PTR et l’enregistrement direct associé correspondent, c’est-à-dire que le nom figurant dans le PTR se résolve à nouveau vers la même adresse IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `-x 192.0.2.10` | Requête inverse : dig construit lui-même le nom PTR correspondant dans la zone `in-addr.arpa` |
| `A mail1.example.com` | Requête directe du nom issu du PTR afin de vérifier que le parcours circulaire mène à la même adresse IP |

</details>

Le nom que votre serveur de messagerie indique dans HELO ou EHLO devrait être le même et également résoluble. Une passerelle qui se présente comme `localhost.localdomain` ou avec un nom interne est moins bien évaluée par les grands destinataires.

Il faut faire preuve de prudence lors de l’ajout d’un enregistrement AAAA. Dès que le serveur de messagerie est joignable et émet via IPv6, les mêmes exigences s’appliquent que pour IPv4, et elles sont même parfois plus strictes. Google exige un PTR valide pour les adresses IPv6 qui envoient des messages. S’il manque, l’envoi est refusé alors qu’il fonctionnait parfaitement via IPv4. Un enregistrement AAAA sur le serveur de messagerie n’est donc jamais une simple modification DNS.

## SPF

SPF définit quels systèmes sont autorisés à envoyer au nom du domaine. L’enregistrement est placé comme TXT sur le domaine lui-même.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### La limite de recherches DNS

L’évaluation d’un enregistrement SPF ne peut déclencher plus de dix mécanismes effectuant des requêtes DNS. Sont comptés `include`, `a`, `mx`, `ptr`, `exists` et `redirect`, de manière récursive : chaque `include` ajoute les recherches de l’enregistrement inclus. Ne sont pas comptés `ip4`, `ip6` et `all`.

Si la limite est dépassée, le résultat est un `permerror`. Pour DMARC, cela signifie un SPF non réussi, même si le serveur émetteur serait en réalité autorisé. Le caractère délicat est que l’erreur survient souvent sans intervention de votre part, parce qu’un fournisseur inclus étend son enregistrement. Votre propre enregistrement n’a pas changé, mais la remise se dégrade malgré tout.

De plus, seuls deux « void lookups », c’est-à-dire des requêtes sans résultat, sont autorisés. Un `include` vers un domaine qui n’existe plus entre dans ce décompte. Les références à des prestataires remplacés doivent donc être supprimées, et non conservées par précaution.

### Ce qui n’a pas sa place dans un enregistrement SPF

- **`ptr`** est certes spécifié, mais est considéré comme obsolète depuis la RFC 7208 et ne doit pas être utilisé. Les systèmes d’évaluation peuvent l’ignorer.
- **`+all`** autorise n’importe quel expéditeur et est donc plus nuisible que l’absence totale d’enregistrement SPF.
- **`?all`** est neutre et n’a donc pratiquement aucune valeur pour DMARC.
- **Un enregistrement séparé de type SPF (type 99)** n’est plus nécessaire. Il a été abandonné depuis la RFC 7208 ; SPF se trouve exclusivement dans TXT.

Entre `~all` (softfail) et `-all` (hardfail), le choix dépend de l’exhaustivité du recensement des chemins d’envoi. Tant qu’il subsiste des doutes, `~all` est le bon choix. Les personnes qui appliquent déjà DMARC et analysent les rapports peuvent passer à `-all`.

### Les sous-domaines n’héritent de rien

Un enregistrement SPF sur `example.com` ne s’applique pas à `newsletter.example.com`. Chaque sous-domaine émetteur a besoin de son propre enregistrement. Pour tous les autres, une entrée générique est recommandée afin d’indiquer clairement qu’aucun envoi ne provient de là :

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attention : un joker TXT répond également aux requêtes concernant des noms tels que `_dmarc.sub.example.com`, pour autant qu’aucun enregistrement explicite n’y existe. C’est généralement sans problème, mais cela peut compliquer le dépannage, car chaque requête TXT reçoit une réponse.

### SPF flattening

Les outils qui résolvent toutes les références `include` et les remplacent par les adresses IP situées derrière résolvent la limite de recherches au prix de la maintenabilité. Si le fournisseur modifie ses adresses, l’envoi échoue, et personne ne s’en aperçoit, car tout semble correct dans son propre enregistrement. Cette approche nécessite donc une comparaison automatisée qui vérifie régulièrement la liste par rapport à la source. En tant que travail manuel ponctuel, cette méthode échoue tôt ou tard.

## DKIM

DKIM signe les messages sortants. La clé publique se trouve sous `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Le sélecteur peut être choisi librement et est défini par le système émetteur. Un nom explicite incluant une date facilite bien davantage la rotation ultérieure que `s1` et `s2`.

### Délégation via CNAME

Lorsque le service d’envoi le propose, la variante CNAME est préférable à l’entrée directe :

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Le fournisseur peut alors effectuer lui-même la rotation de sa clé sans que personne n’ait à intervenir dans votre zone. Cette rotation est autrement régulièrement négligée, car elle exige une coordination entre deux équipes. Un CNAME exclut toutefois tout autre enregistrement au même nom ; c’est une règle fondamentale du DNS, et non une particularité de DKIM.

### Rotation sans interruption

Lors d’un changement de clé, le nouveau sélecteur est d’abord publié, puis le serveur émetteur bascule vers celui-ci et l’ancien enregistrement n’est supprimé qu’ensuite. Supprimer immédiatement l’ancienne clé invalide les signatures de tous les messages encore en transit ou dans des files d’attente et rend les vérifications ultérieures impossibles. Un délai de quelques jours entre le basculement et la suppression est approprié.

Un enregistrement dont le `p=` est vide n’est d’ailleurs pas une entrée défectueuse, mais la manière spécifiée d’indiquer qu’une clé a été retirée.

### Longueur de clé

1024 bits sont considérés comme obsolètes, 2048 bits sont la norme. Des clés RSA plus grandes n’apportent pratiquement aucun avantage supplémentaire et augmentent seulement le risque qu’un système intermédiaire ne traite pas correctement l’enregistrement.

## DMARC

DMARC relie SPF et DKIM à une instruction indiquant quoi faire lorsqu’une vérification échoue, et renvoie des rapports. L’enregistrement se trouve sous `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Depuis mai 2026, la version révisée définie par la RFC 9989 ainsi que les spécifications de rapport RFC 9990 et RFC 9991 remplace la RFC 7489. Trois changements sont importants en pratique :

- **`pct` a été supprimé.** L’introduction progressive via un pourcentage n’existe plus. Il est remplacé par `t=y`, qui désigne le domaine comme étant en phase de test : les rapports continuent, mais la politique ne doit pas être appliquée.
- **`np` est nouveau.** Il définit la politique pour les sous-domaines inexistants et comble ainsi une lacune que les attaquants exploitent volontiers, car les sous-domaines inventés n’étaient jusqu’ici couverts que par `sp`. Sans indication propre, `np` suit la valeur de `sp`.
- **La Public Suffix List est remplacée par un `Tree Walk`.** Le domaine organisationnel n’est plus déterminé à partir d’une liste gérée de manière externe, mais par des requêtes DNS graduelles le long de l’arborescence des noms. Cela modifie sensiblement l’évaluation pour les grands espaces de noms comportant de nombreux niveaux.

### L’alignement est le véritable cœur du mécanisme

DMARC ne réussit pas parce que SPF ou DKIM ont techniquement réussi, mais seulement si au moins l’un des deux correspond également au domaine expéditeur visible dans l’en-tête `From`. SPF est vérifié par rapport au domaine de l’expéditeur d’enveloppe, qui diffère régulièrement lors des redirections, des services de newsletters et des systèmes de tickets. C’est précisément pourquoi des messages avec un SPF valide ne passent parfois pas la vérification DMARC.

Avec `adkim=r` et `aspf=r` (relaxed, la valeur par défaut), une correspondance au niveau du domaine organisationnel suffit. `s` exige une égalité exacte, sous-domaine inclus, et échoue en pratique presque toujours sur l’un des chemins d’envoi.

### Les adresses de rapport externes nécessitent une autorisation

Si les rapports doivent être envoyés à une adresse en dehors de votre propre domaine, par exemple à un service d’analyse DMARC, le domaine destinataire doit l’autoriser. Sans cet enregistrement, de nombreux destinataires n’envoient tout simplement rien, et l’analyse reste vide alors que tout paraît correct dans votre propre enregistrement :

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Cette entrée est créée par l’opérateur de la zone de destination, et non par vous. Pour les services commerciaux, cela se fait automatiquement, mais pas pour une boîte de collecte exploitée par vos soins dans un autre domaine qui vous appartient.

### Erreurs de syntaxe typiques

Les noms de balise et les valeurs de politique doivent être écrits en minuscules ; `p=Reject` n’est pas valide. Les balises sont séparées par un point-virgule ; l’absence de séparateur rend le reste de la ligne inopérant. De plus, `p` doit être la première balise après `v`. Un enregistrement composé uniquement de `v=DMARC1; rua=...` ne contient aucune politique et est incomplet.

### Le déploiement

`p=none` est un état de mesure, pas un objectif. Il ne modifie pas le traitement de vos e-mails par les destinataires et sert uniquement à identifier tous les chemins d’envoi légitimes au moyen des rapports. Si, après l’introduction, vous ne passez pas dans les quelques mois de `quarantine` à `reject`, vous avez fourni l’effort sans obtenir la protection. L’aspect organisationnel de cette démarche, y compris le document d’aide à la décision, constitue un sujet à part entière et est décrit dans le blueprint DMARC.

## MTA-STS et TLS-RPT

SMTP chiffre de manière opportuniste : si le système distant propose STARTTLS, la connexion est chiffrée ; sinon, elle ne l’est pas. Un attaquant en mesure de manipuler le trafic peut supprimer l’annonce STARTTLS et maintenir ainsi la connexion en clair. MTA-STS comble cette lacune pour les domaines destinataires.

MTA-STS se compose de deux parties, dont une seule se trouve dans le DNS :

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La politique proprement dite se trouve sous forme de fichier à l’adresse `https://mta-sts.example.com/.well-known/mta-sts.txt` et doit être fournie via un certificat valide :

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Les sources d’erreurs se trouvent presque toutes en dehors de la zone :

- **L’`id` doit changer à chaque modification de politique.** C’est la seule indication permettant aux systèmes émetteurs de savoir qu’ils doivent récupérer une nouvelle politique. Quiconque modifie le fichier mais laisse l’`id` inchangé travaille contre des copies mises en cache jusqu’à l’expiration de `max_age`.
- **La liste MX de la politique et les enregistrements MX doivent correspondre.** Un nouveau MX absent de la politique est refusé par les expéditeurs utilisant `mode: enforce`. Lors des migrations, la politique doit donc être adaptée avant le changement des MX.
- **`mode: testing` en premier.** Dans ce mode, les violations sont seulement signalées, et non appliquées. Le passage à `enforce` a lieu lorsque les rapports sont propres.
- **Un enregistrement CAA peut bloquer l’émission du certificat pour l’hôte de politique**, s’il indique une autorité de certification différente de celle utilisée.

TLS-RPT fournit les rapports associés et consiste en un seul enregistrement :

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT est également utile sans MTA-STS, car il rend enfin visibles les échecs du chiffrement de transport.

## DANE

DANE atteint le même objectif que MTA-STS, mais ancre la confiance dans le DNS plutôt que dans la PKI Web. Il nécessite une zone intégralement signée avec DNSSEC ; sans DNSSEC, un enregistrement TLSA est inefficace.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Point essentiel en exploitation : à chaque changement de certificat, l’enregistrement TLSA doit correspondre au préalable. La procédure habituelle publie le nouveau hachage en parallèle de l’ancien, change ensuite le certificat, puis supprime l’ancienne entrée. Inverser cet ordre rend le serveur de messagerie inaccessible à tous les expéditeurs vérifiant DANE, parmi lesquels figurent notamment les grands fournisseurs germanophones. En Suisse, DANE est nettement moins répandu que MTA-STS, principalement en raison de l’absence de signature DNSSEC de la zone.

## BIMI

BIMI affiche le logo de la marque dans la boîte de réception et est le seul mécanisme traité ici qui ne soit pas encore une RFC, mais qui reste un Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

Les exigences sont élevées : une politique DMARC appliquée avec `quarantine` ou `reject`, un logo au format SVG Tiny Portable/Secure et, pour la plupart des fournisseurs, un Verified Mark Certificate payant. BIMI n’est donc pas un mécanisme de sécurité, mais une question de visibilité, et doit venir à la fin de l’ordre des priorités, pas au début.

## Autres enregistrements connexes

**Autodiscover et SRV :** les environnements Exchange utilisent `autodiscover.example.com` comme CNAME ou un enregistrement SRV `_autodiscover._tcp.example.com`. Tous deux concernent la configuration des clients et non le flux de messagerie, mais sont volontiers oubliés lors des migrations et entraînent alors des profils qui ne peuvent plus être configurés.

**CAA :** n’a pas de lien direct avec la messagerie, mais détermine quelle autorité de certification peut émettre un certificat pour `mta-sts.example.com` ou le nom du serveur de messagerie.

**Zones split-horizon :** lorsqu’une zone DNS interne porte le même nom que la zone publique, les enregistrements de messagerie n’existent souvent pas en interne. Les systèmes internes qui effectuent une vérification SPF ou DKIM obtiennent alors des résultats différents de ceux du monde extérieur. Toute modification d’enregistrements de messagerie doit donc s’accompagner de la question de savoir si la zone interne doit être mise à jour.

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

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `@1.1.1.1` | Envoie la requête à ce résolveur au lieu de celui configuré dans `/etc/resolv.conf` |
| `+short` | N’affiche que les valeurs des enregistrements, sans en-têtes ni métadonnées |
| `MX`, `TXT` | Types d’enregistrements interrogés |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | Les noms définis sous le domaine pour DMARC, DKIM, MTA-STS et TLS-RPT |

</details>

Contre le serveur faisant autorité, afin de contourner entièrement le cache :

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
| `+norecurse` | Ne définit pas le bit Recursion Desired ; le serveur répond uniquement à partir de ses propres données de zone, et non depuis un cache |

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
| `1.1.1.1` | Résolveur à utiliser au lieu de celui configuré à l’échelle du système |

</details>

Pour une évaluation complète, y compris le comptage des recherches SPF, la recherche de sélecteurs DKIM et la vérification d’alignement, cette page propose le [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), qui vérifie un domaine en une seule fois par rapport à tous les enregistrements décrits ici.

Le test le plus révélateur reste toutefois un véritable message. Envoyez un e-mail à une boîte aux lettres auprès d’un grand fournisseur et examinez la ligne `Authentication-Results` dans les en-têtes. Elle indique en une ligne les résultats réels de SPF, DKIM et DMARC et remplace toute théorie concernant le fichier de zone.

## Ordre des étapes lors d’une migration

Lors d’un changement de fournisseur de messagerie, cette séquence a fait ses preuves :

1. Abaisser le TTL de tous les enregistrements concernés à 300 secondes, au moins un jour à l’avance.
2. Publier les sélecteurs DKIM du nouveau fournisseur tant que les anciens sont encore présents.
3. Étendre SPF avec le nouveau fournisseur sans retirer l’ancien, et recalculer la limite de recherches.
4. Pour MTA-STS, adapter la politique aux nouveaux noms MX et augmenter l’`id` avant le changement des enregistrements MX.
5. Modifier les MX et surveiller la remise.
6. Supprimer les anciens includes SPF et sélecteurs DKIM seulement après quelques jours sans incident.
7. Rétablir le TTL.

Le problème le plus fréquent dans cette séquence est une étape 6 exécutée trop tôt : les anciennes entrées sont supprimées en même temps que le basculement, et tout ce qui passe encore par l’ancien chemin échoue lors de la vérification de l’expéditeur.

## Conclusion

Les enregistrements de messagerie se distinguent de toutes les autres entrées DNS en ce qu’une erreur n’est pas forcément visible. Un enregistrement A erroné entraîne un ticket en quelques minutes ; un enregistrement SPF en double ou une clé DKIM comportant un caractère en trop provoque en revanche un taux de remise qui diminue lentement durant des semaines.

Trois règles évitent la plupart de ces situations. Premièrement : avant tout nouvel enregistrement, vérifier ce qui existe déjà au lieu d’en ajouter un deuxième à côté. Deuxièmement : après chaque modification, contrôler auprès d’un résolveur public et comparer la valeur caractère par caractère au modèle, pas seulement visuellement. Troisièmement : lors des changements, toujours publier d’abord le nouveau, basculer ensuite, puis supprimer l’ancien. En respectant cet ordre, vous disposez toujours d’une solution de repli pour les enregistrements de messagerie.

## Sources

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Définit notamment la limite de 255 octets d’une seule `character-string` dans les enregistrements TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Indique dans la section 10.3 que la cible d’un enregistrement MX ne peut pas être un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Limite de recherches de dix mécanismes, limite des recherches sans résultat, suppression du type RR SPF et recommandation de ne pas utiliser le mécanisme `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Structure de l’enregistrement de clé sous `_domainkey`, rôle du sélecteur et du `p=` vide.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Spécification DMARC actuelle de mai 2026, remplace la RFC 7489 ; suppression de `pct`, nouvelle balise `np`, Tree Walk au lieu de la Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Format et remise des rapports agrégés, y compris l’autorisation des domaines destinataires externes.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Marquage des domaines qui n’acceptent pas d’e-mails.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Enregistrement DNS, fichier de politique, rôle de l’`id` et des modes `testing` et `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Structure de l’enregistrement `_smtp._tls` et des rapports.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Enregistrements TLSA pour SMTP et condition préalable d’une zone signée DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): État actuel de la spécification BIMI, toujours pas une RFC.

12.  [Google: Directives pour les expéditeurs d’e-mails](https://support.google.com/a/answer/81126): Exigences pour les expéditeurs, notamment l’obligation d’un PTR pour les adresses IPv6 émettrices et les règles applicables aux expéditeurs de masse depuis février 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Exigences pour les expéditeurs à partir de 5 000 messages par jour, en vigueur depuis mai 2025.
