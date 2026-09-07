---
title: "Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu"
navTitle: "Ghost Sender"
description: "La remise directe à Exchange Online contourne une passerelle en amont si le tenant ne la bloque pas explicitement. Le risque est réel, mais la cause réside dans une configuration de flux de messagerie incomplète."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: 6a500f1ed53a180322afb3c86e44376100d68659eeb55ffae35937ab434c6b61
translatedAt: 2026-09-05T07:48:11.305Z
url: https://rafaelpfister.ch/fr/blog/ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu
translationModel: gpt-5.6-terra
---

# Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu

![Un administrateur fantôme maintient ouverte, dans le centre de données, la porte située à côté du portail de sécurité, tandis que les e-mails parviennent directement dans la boîte aux lettres en contournant le filtre.](../images/ghost-admin.png)

La possibilité d’attaque décrite par InfoGuard Labs sous le nom de « Ghost Sender » est réelle : un attaquant peut contourner une passerelle e-mail en amont et remettre directement des messages à Exchange Online. Toutefois, cela suppose que le tenant accepte toujours ce chemin direct. Il ne s’agit pas d’une vulnérabilité universelle d’Exchange Online, mais d’une topologie de flux de messagerie insuffisamment sécurisée.

Un agent de transfert de courrier qui héberge les boîtes aux lettres d’un domaine accepte par principe les connexions SMTP provenant d’Internet. L’enregistrement MX indique aux expéditeurs légitimes le chemin de remise souhaité. Il n’est ni une règle de pare-feu ni une liste d’accès et n’empêche personne de s’adresser directement à un point de terminaison Exchange Online connu.

## Ce que montre réellement « Ghost Sender »

Le [scénario décrit par InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) est le suivant :

1. Une organisation héberge ses boîtes aux lettres dans Exchange Online.
2. L’enregistrement MX public pointe vers une passerelle de messagerie sécurisée en amont.
3. Le point de terminaison Exchange Online sous `*.mail.protection.outlook.com` reste directement accessible depuis Internet.
4. L’administrateur n’a pas restreint Exchange Online afin que seule la passerelle en amont puisse y remettre des messages.
5. Un attaquant ignore l’enregistrement MX et remet directement son message à Exchange Online.

Le chemin prévu est donc :

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Mais ce chemin est resté ouvert :

```text
Angreifer -> Exchange Online -> Postfach
```

Il s’agit d’une mauvaise configuration à prendre au sérieux. Le filtre en amont peut être contourné par cette voie ; l’usurpation d’expéditeur, le phishing et la fraude au président s’en trouvent considérablement facilités. InfoGuard mérite d’être salué pour avoir rendu ce problème visible, étudié son ampleur et publié un test facile à utiliser.

Mais où se situerait précisément le défaut du produit ?

La dramatisation médiatique aide peu à comprendre la situation. [Heise titre qu’Exchange Online laisse passer des e-mails falsifiés « sans broncher »](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), alors que seules certaines configurations tierces et hybrides insuffisamment renforcées sont concernées. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) l’exprime bien plus justement : ce n’est pas une faille de sécurité au sens strict, mais un problème de conception et de configuration.

## « An MTA is doing MTA-Things »

Chaque tenant Exchange Online possède un point de terminaison SMTP public. Ce point de terminaison n’est pas secret et ne doit pas l’être. Microsoft explique lui-même qu’Exchange Online accepte par défaut les messages adressés directement aux boîtes aux lettres qui y sont hébergées : [c’est tout simplement le fonctionnement de l’e-mail](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

[SMTP lui-même décrit également l’enregistrement MX comme un mécanisme permettant d’identifier le système cible habituel](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Il n’en découle aucune obligation pour le serveur cible de rejeter les connexions passant par tout autre hôte accessible. Un attaquant n’est pas tenu de suivre le chemin balisé. Si un autre MTA est accessible, connaît le domaine destinataire et accepte le message, il sera essayé, tout comme les spammeurs tentent depuis des décennies de joindre des systèmes MX de secours moins bien protégés.

Lorsqu’on place un filtre tiers en amont, on modifie la topologie standard. « Exchange Online est ma passerelle de messagerie Internet » devient « seule ma passerelle tierce est autorisée à transmettre les e-mails Internet à Exchange Online ». Cette nouvelle `Trust-Border` ne naît pas d’une entrée DNS. Elle doit être explicitement imposée au système destinataire.

C’est exactement ce que documente Microsoft : avec un MX externe, il faut créer un connecteur entrant de type `Partner` qui, pour `SenderDomains *`, n’accepte que le certificat ou les adresses IP source du service en amont. Les messages remis directement en contournant la passerelle sont alors rejetés. Cela figure tel quel dans le guide Microsoft [« Manage mail flow using a third-party cloud service with Exchange Online »](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius décrit également en détail cette « entrée secondaire » dans la [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM et DMARC ne sont pas des videurs

InfoGuard montre des messages pour lesquels SPF, DKIM et DMARC échouent et qui arrivent néanmoins dans la boîte aux lettres. Cela paraît spectaculaire, mais il ne s’agit pas d’un « contournement » cryptographique de ces mécanismes. Les e-mails ne réussissent justement pas ces contrôles. Ils délivrent `fail`. Ce qui compte est l’action locale que le système destinataire déduit de ce résultat.

SPF vérifie si un système est autorisé à envoyer pour l’expéditeur de l’enveloppe. DKIM vérifie une signature. DMARC associe ces résultats au domaine d’expéditeur visible et publie un traitement souhaité. Même la [norme DMARC actuelle RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) précise explicitement que le destinataire peut tenir compte de ce traitement souhaité, mais n’y est pas obligé. DMARC est un signal important, mais pas un contrôle d’accès réseau.

Avec une passerelle en amont, Exchange Online voit d’abord l’adresse IP de cette passerelle, et non celle de l’expéditeur initial. C’est à cela que sert [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : il reconstruit la source d’origine et améliore les évaluations SPF, DKIM, DMARC, anti-usurpation et anti-phishing. Mais Enhanced Filtering n’est pas non plus une serrure. Il ne remplace pas le connecteur partenaire restrictif.

La mauvaise configuration devient particulièrement évidente lorsqu’un administrateur affaiblit ou contourne complètement le contrôle EOP via un contournement SCL, parce que le produit en amont est censé déjà filtrer, tout en laissant ouverte la remise directe depuis Internet. Dans ce cas, il n’a pas subi le « contournement » d’un mécanisme de protection : il n’a volontairement plus prévu de protection efficace pour l’une des deux entrées.

On peut tout à fait reprocher à Microsoft qu’un message présentant une erreur d’authentification clairement visible arrive dans la boîte de réception sans avertissement. On peut critiquer la sémantique des types de connecteurs, la documentation et l’absence d’avertissements dans le Configuration Analyzer. Ce sont tous des points légitimes. L’existence d’un point de terminaison SMTP publiquement accessible n’est toutefois pas une faille de sécurité.

## « Direct Send » n’est pas synonyme de « remise directe »

Deux choses sont confondues dans la discussion :

- **Direct Send** désigne chez Microsoft les messages anonymes dont l’expéditeur de l’enveloppe (`5321.MailFrom`) utilise un domaine accepté propre au tenant.
- **La remise directe à Exchange Online** désigne de façon générale un message SMTP qui ignore le MX tiers publié et est remis directement au point de terminaison Exchange. L’expéditeur peut également utiliser n’importe quel domaine externe.

Direct Send possède son propre commutateur :

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-RejectDirectSend $true` | Rejette les remises directes anonymes dont l’expéditeur de l’enveloppe utilise un domaine accepté du tenant |

</details>

Ce commutateur est utile lorsque Direct Send n’est pas nécessaire. Il empêche l’usurpation d’un domaine interne par ce chemin. Il ne ferme toutefois pas toute l’entrée secondaire aux expéditeurs externes arbitraires. Microsoft décrit le champ d’application exact dans la [documentation du cmdlet `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Pour empêcher complètement « Ghost Sender », il faut toujours une restriction d’accès via un connecteur partenaire ou une règle de flux de messagerie appropriée.

## Microsoft doit-il vraiment tout faire à la place de l’administrateur ?

Non. Quiconque ajoute un filtre de messagerie supplémentaire à une chaîne de transport de production assume la responsabilité de cette chaîne de transport.

Le fournisseur ne peut pas deviner de manière fiable si, en plus du MX externe, des scanners, appareils multifonctions, services SaaS, serveurs hybrides, relais partenaires ou d’autres systèmes légitimes doivent envoyer directement vers Exchange Online. Un blocage automatique du type « le MX pointe ailleurs, donc je bloque tout le reste » interromprait des flux de messagerie souhaités dans de nombreux environnements réels. L’administrateur doit donc définir explicitement la limite de confiance souhaitée.

Microsoft peut néanmoins faciliter la tâche des responsables. Un bon Configuration Analyzer devrait détecter un MX externe sans connecteur partenaire restrictif et émettre un avertissement clair. L’assistant de configuration pourrait expliquer qu’un connecteur de type « Votre organisation » identifie certes les connexions appropriées, mais ne rejette pas automatiquement les connexions inappropriées. Des commutateurs sécurisés par défaut et de meilleurs rapports d’exploitation seraient également bienvenus.

Ce serait un renforcement utile du produit. Cela ne change toutefois rien à l’évaluation technique : une topologie spéciale non sécurisée reste une configuration non sécurisée et ne devient pas un zero-day uniquement parce qu’elle est largement répandue.

## Comment fermer l’entrée secondaire

Pour les environnements dotés d’un filtre en amont, les points suivants doivent au minimum figurer sur la liste de contrôle :

1. **Documenter intégralement le flux de messagerie.** Quels systèmes sont réellement autorisés à remettre des messages à Exchange Online ? Cela inclut également les chemins hybrides, applicatifs et de secours.
2. **Configurer un connecteur partenaire restrictif.** Utiliser `SenderDomains *` et limiter la remise à un certificat (de préférence) ou à des plages d’adresses IP source maintenues. Un connecteur de type `OnPremises` ou « Votre organisation » n’impose pas cet effet de refus par défaut (voir par exemple : [Routage de messagerie entre Apache James et Exchange Online](/blog/totemomail-m365)).
3. **Configurer correctement Enhanced Filtering.** Si EOP doit continuer à filtrer, l’IP d’origine et les informations de l’expéditeur doivent être correctement reconstruites. Les contournements SCL-`-1` généralisés doivent être examinés de manière critique.
4. **Désactiver Direct Send s’il n’est pas utilisé.** Vérifier au préalable, au moyen de Message Trace ou des rapports disponibles, si des scanners ou applications en dépendent.
5. **Ne pas basculer aveuglément.** Tester puis surveiller les plages IP de la passerelle, les changements de certificat, le flux de messagerie hybride ainsi que les chemins particuliers `onmicrosoft.com`-, Teams et autres.

Un exemple simplifié de la variante basée sur les adresses IP est le suivant :

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-Name` | Nom d’affichage du nouveau connecteur entrant |
| `-ConnectorType Partner` | Classe de connecteur pour les systèmes partenaires externes ; seul ce type impose le rejet des connexions inappropriées |
| `-SenderDomains *` | Le connecteur s’applique au courrier provenant de tous les domaines expéditeurs |
| `-RestrictDomainsToIPAddresses $true` | Active le blocage : les e-mails des domaines indiqués ne sont plus acceptés que depuis les adresses figurant dans `-SenderIpAddresses` |
| `-SenderIpAddresses` | Les adresses ou plages IP source autorisées de la passerelle en amont |
| `-RequireTls $true` | Exige le chiffrement TLS pour les connexions via ce connecteur |

</details>

Lorsque c’est possible, la liaison par certificat est préférable à une liste d’autorisation IP. Les modifications doivent d’abord être effectuées dans un test contrôlé, car une liste d’autorisation erronée transforme très vite l’entrée secondaire ouverte en panne complète de messagerie.

## Le test simple à effectuer soi-même

Le test présenté par InfoGuard (et la MSXFAQ) est utile :

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-SmtpServer` | Hôte cible : le point de terminaison Exchange Online public du tenant, délibérément en contournant le MX |
| `-To` | Adresse du destinataire dans le tenant à tester |
| `-From` | Adresse d’expéditeur externe quelconque ; c’est précisément ce que l’entrée secondaire ne devrait plus accepter |
| `-Subject` | Ligne d’objet, pour la retrouver dans Message Trace |
| `-Body` | Corps du message |

</details>

Avec un connecteur partenaire correctement restreint, il faut s’attendre à un rejet SMTP tel que `5.7.51 TenantInboundAttribution; Rejecting`. Une règle de transport alternative peut d’abord accepter le message puis le déplacer en quarantaine ; il faut donc contrôler non seulement la réponse SMTP, mais aussi Message Trace, la quarantaine et la boîte aux lettres. `Send-MailMessage` (obsolète) ne sert ici qu’à fournir une illustration facile à comprendre. Tout outil de test SMTP contrôlé remplit le même objectif.

## Un test utile avec une étiquette trompeuse

« Ghost Sender » n’est pas un nouvel exploit SMTP. C’est un nom accrocheur pour une entrée secondaire ouverte, dont Microsoft documente depuis longtemps la sécurisation et que l’administrateur a laissée ouverte.

L’ironie est qu’InfoGuard qualifie lui-même le problème, dans sa propre contribution, de « widespread and systematic misconfiguration » et conclut par la phrase « Ghost-Sender is a misconfiguration ». Le Security Response Center de Microsoft n’a pas non plus initialement considéré le signalement comme une vulnérabilité. Les faits sont donc bien présents dans l’article : seul le titre, l’e-mail de test et le branding « Vulnerability » suggèrent malheureusement une interprétation plus dramatique.

La partie utile de la publication est l’appel à la vigilance : de nombreuses entreprises n’ont manifestement pas correctement verrouillé leur flux de messagerie. La partie problématique est l’affirmation selon laquelle Exchange Online présenterait pour cette raison une faille de sécurité universelle. Non : Exchange Online se comporte d’abord ici comme un MTA. Ce qui le rend non sécurisé, c’est une limite de confiance dont la configuration n’a pas été menée à terme.

Microsoft doit-il vraiment tout faire à la place de l’administrateur ? Non. Mais il faut manifestement rappeler sans cesse que le routage DNS ne remplace pas le contrôle d’accès.

## Sources

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/) : L’étude initiale, avec l’analyse de sa diffusion et sa propre conclusion : « Ghost-Sender is a misconfiguration ».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/) : Le test en ligne publié par InfoGuard pour vérifier si son propre tenant possède une entrée secondaire ouverte.

3.  [MSXFAQ: Exchange Online comme entrée secondaire pour la réception d’e-mails](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) : L’évaluation de Frank Carius : pas une erreur dans Exchange Online, mais une mauvaise configuration de l’administrateur.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865) : Microsoft explique que l’acceptation directe de messages destinés à des boîtes aux lettres hébergées correspond au fonctionnement de l’e-mail, et distingue Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud) : Le guide officiel incluant sa propre étape relative au connecteur partenaire restrictif en cas de MX externe.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : Reconstruit la source d’expéditeur d’origine derrière une passerelle ; améliore l’évaluation, mais ne remplace pas le connecteur.

7.  [Heise: Ghost-Sender – Exchange Online laisse passer sans broncher des e-mails falsifiés](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html) : Exemple de couverture sensationnaliste qui généralise certaines mauvaises configurations.

8.  [Crow in the Cloud: Les fantômes que je n’ai pas appelés](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) : Une évaluation pertinente comme problème de conception et de configuration, avec des mesures de protection.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html) : Décrit l’enregistrement MX comme un mécanisme permettant d’identifier le système cible habituel, et non comme un contrôle d’accès.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html) : Précise que le destinataire peut tenir compte du traitement DMARC publié, mais n’y est pas obligé.

---

## Votre flux de messagerie est-il sécurisé ?

Vous ne savez pas si votre tenant Exchange Online possède lui aussi une entrée secondaire ouverte ? **adeptio** vérifie l’ensemble de votre flux de messagerie : des enregistrements MX, connecteurs et passerelles tierces à EOP, SPF, DKIM, DMARC et Direct Send. Avec une approche pratique, indépendante et des recommandations concrètes.

Pour faire vérifier ou sécuriser correctement votre flux de messagerie, vous pouvez volontiers convenir d’un entretien de conseil sans engagement :

**[Réserver un entretien de conseil avec adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
