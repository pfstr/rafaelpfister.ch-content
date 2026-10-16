---
title: "Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu"
navTitle: "Ghost Sender"
description: "La remise directe à Exchange Online contourne une passerelle en amont lorsque le tenant ne la bloque pas explicitement. Le risque est réel, mais sa cause réside dans une configuration incomplète du flux de messagerie."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: da918db49e8e0b5243e6a3c354324682ddb6bda54ec743be474f3011df3f9dc6
translatedAt: 2026-10-14T08:54:13.565Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/fr/blog/ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu
---

# Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu

![Un administrateur fantôme maintient ouverte, dans le centre de données, la porte située à côté du portail de sécurité, tandis que des e-mails parviennent directement dans la boîte aux lettres en contournant le filtre.](../images/ghost-admin.png)

La possibilité d’attaque décrite par InfoGuard Labs sous le nom de « Ghost Sender » est réelle : un attaquant peut contourner une passerelle de messagerie en amont et remettre directement des messages à Exchange Online. Cela suppose toutefois que le tenant accepte toujours cette voie directe. Il ne s’agit pas d’une vulnérabilité universelle d’Exchange Online, mais d’une topologie de flux de messagerie insuffisamment sécurisée.

Un agent de transfert de courrier qui gère les boîtes aux lettres d’un domaine accepte par principe des connexions SMTP depuis Internet. L’enregistrement MX indique aux expéditeurs légitimes le chemin de remise souhaité. Il n’est ni une règle de pare-feu ni une liste de contrôle d’accès, et n’empêche personne de s’adresser directement à un point de terminaison Exchange Online connu.

## Ce que montre réellement « Ghost Sender »

Le scénario [décrit par InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) se présente ainsi :

1. Une organisation héberge ses boîtes aux lettres dans Exchange Online.
2. L’enregistrement MX public pointe vers une Secure Email Gateway en amont.
3. Le point de terminaison Exchange Online sous `*.mail.protection.outlook.com` reste directement accessible depuis Internet.
4. L’administrateur n’a pas restreint Exchange Online afin que seule la passerelle en amont puisse y remettre des messages.
5. Un attaquant ignore l’enregistrement MX et remet son message directement à Exchange Online.

Le chemin prévu est donc le suivant :

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Mais ce chemin reste ouvert :

```text
Angreifer -> Exchange Online -> Postfach
```

Il s’agit d’une erreur de configuration sérieuse. Le filtre en amont peut être contourné par cette voie ; les expéditeurs usurpés, le phishing et la fraude au président s’en trouvent considérablement facilités. InfoGuard mérite d’être félicité d’avoir rendu ce problème visible, étudié son ampleur et publié un test simple d’utilisation.

Mais où se situerait précisément le défaut du produit ?

La dramatisation médiatique aide elle aussi peu à situer le problème. [Heise titre qu’Exchange Online laisse passer les e-mails falsifiés « sans difficulté »](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), alors que seules certaines configurations tierces et hybrides incomplètement renforcées sont concernées. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) le formule bien plus justement : ce n’est pas une faille de sécurité au sens strict, mais un problème de conception et de configuration.

## « An MTA is doing MTA-Things »

Chaque tenant Exchange Online dispose d’un point de terminaison SMTP public. Ce point de terminaison n’est pas secret et ne doit pas l’être. Microsoft explique lui-même qu’Exchange Online accepte par défaut les messages adressés directement aux boîtes aux lettres qui y sont hébergées : [c’est tout simplement ainsi que fonctionne le courrier électronique](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

De même, [SMTP décrit lui-même l’enregistrement MX comme un mécanisme permettant d’identifier le système de destination habituel](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Il n’en découle aucune obligation pour le serveur de destination de refuser les connexions passant par tout autre hôte accessible. Un attaquant n’est pas tenu de suivre le chemin indiqué. Si un autre MTA est accessible, connaît le domaine destinataire et accepte le message, il sera essayé, tout comme les spammeurs tentent depuis des décennies de contacter des systèmes MX de secours moins bien protégés.

Quiconque place un filtre tiers en amont modifie la topologie standard. De « Exchange Online est ma passerelle de messagerie Internet », on passe à « seule ma passerelle tierce peut transmettre du courrier Internet à Exchange Online ». Cette nouvelle `Trust-Border` ne naît pas d’une entrée DNS. Elle doit être explicitement imposée au système destinataire.

C’est exactement ce que documente Microsoft : avec un MX externe, il convient de créer un connecteur entrant de type `Partner` qui, pour `SenderDomains *`, n’accepte que le certificat ou les adresses IP sources du service en amont. Les messages remis directement en contournant la passerelle sont alors rejetés. Cela figure tel quel dans les instructions Microsoft [« Manage mail flow using a third-party cloud service with Exchange Online »](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius décrit également en détail cette « entrée latérale » dans la [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM et DMARC ne sont pas des videurs

InfoGuard montre des messages pour lesquels SPF, DKIM et DMARC échouent, mais qui arrivent malgré tout dans la boîte aux lettres. Cela paraît spectaculaire, mais ce n’est pas un « contournement » cryptographique de ces mécanismes. Les e-mails ne passent justement pas avec succès. Ils fournissent `fail`. L’élément déterminant est l’action locale que le système destinataire déduit de ce résultat.

SPF vérifie si un système est autorisé à envoyer pour l’expéditeur d’enveloppe. DKIM vérifie une signature. DMARC relie ces résultats au domaine d’expéditeur visible et publie le traitement souhaité. Même la [norme DMARC actuelle, RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1), précise expressément que le destinataire peut prendre en compte ce traitement souhaité, sans y être obligé. DMARC est un signal important, mais pas un contrôle d’accès réseau.

Avec une passerelle en amont, Exchange Online voit en outre d’abord l’adresse IP de cette passerelle, et non celle de l’expéditeur d’origine. Il existe pour cela [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : cette fonction reconstitue la source d’origine et améliore les évaluations SPF, DKIM, DMARC, anti-usurpation et anti-phishing. Mais Enhanced Filtering n’est pas non plus une serrure. Il ne remplace pas le connecteur partenaire restrictif.

La mauvaise configuration devient particulièrement évidente lorsqu’un administrateur affaiblit ou contourne entièrement la vérification EOP via un bypass SCL, puisque le produit en amont est censé déjà filtrer, tout en laissant ouverte la remise directe depuis Internet. Dans ce cas, il ne s’est pas fait « contourner » un mécanisme de protection : il a délibérément cessé de prévoir une protection efficace pour l’une des deux entrées.

On peut tout à fait reprocher à Microsoft qu’un message présentant une erreur d’authentification clairement visible arrive dans la boîte de réception sans avertissement. On peut critiquer la sémantique des types de connecteurs, la documentation et l’absence d’avertissements dans Configuration Analyzer. Tous ces points sont légitimes. L’existence d’un point de terminaison SMTP publiquement accessible n’est toutefois pas une faille de sécurité.

## « Direct Send » n’est pas synonyme de « remise directe »

La discussion mélange deux notions :

- **Direct Send** désigne chez Microsoft des messages anonymes dont l’expéditeur d’enveloppe (`5321.MailFrom`) utilise un domaine accepté propre au tenant.
- **La remise directe à Exchange Online** désigne de manière générale un message SMTP qui ignore le MX tiers publié et est remis directement au point de terminaison Exchange. L’expéditeur peut également utiliser n’importe quel domaine externe.

Direct Send dispose de son propre commutateur :

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Options expliquées</summary>

| Option | Effet |
|---|---|
| `-RejectDirectSend $true` | Rejette les remises directes anonymes dont l’expéditeur d’enveloppe utilise un domaine accepté du tenant |

</details>

Ce commutateur est utile lorsque Direct Send n’est pas nécessaire. Il empêche l’usurpation de domaines internes par cette voie. Il ne ferme toutefois pas toute l’entrée latérale pour des expéditeurs externes quelconques. Microsoft décrit précisément son champ d’application dans la [documentation du cmdlet `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Pour empêcher complètement « Ghost Sender », il faut toujours une restriction d’accès via un connecteur partenaire ou une règle de flux de messagerie appropriée.

## Microsoft doit-il vraiment tout faire à la place de l’administrateur ?

Non. Quiconque ajoute un filtre de messagerie supplémentaire à une chaîne de transport de production assume la responsabilité de cette chaîne.

Le fournisseur ne peut pas deviner de façon fiable si, en plus du MX externe, des scanners, appareils multifonctions, services SaaS, serveurs hybrides, relais partenaires ou d’autres systèmes légitimes doivent envoyer directement à Exchange Online. Un mécanisme automatique du type « le MX pointe ailleurs, donc je bloque tout le reste » interromprait des flux de messagerie souhaités dans de nombreux environnements réels. L’administrateur doit donc définir explicitement la limite de confiance souhaitée.

Microsoft peut néanmoins faciliter la tâche aux responsables. Un bon Configuration Analyzer devrait détecter un MX externe sans connecteur partenaire restrictif et émettre un avertissement clair. L’assistant de configuration pourrait expliquer qu’un connecteur de type « Votre organisation » identifie certes les connexions appropriées, mais ne rejette pas automatiquement les connexions inappropriées. Des commutateurs sécurisés par défaut et de meilleurs rapports opérationnels seraient également bienvenus.

Ce serait un durcissement produit pertinent. Cela ne change toutefois rien à l’évaluation technique : une topologie spéciale non sécurisée reste une configuration non sécurisée et ne devient pas un zero-day du seul fait de sa large diffusion.

## Comment fermer l’entrée latérale

Pour les environnements dotés d’un filtre en amont, les points suivants doivent au minimum figurer dans la liste de contrôle :

1. **Documenter intégralement le flux de messagerie.** Quels systèmes sont réellement autorisés à remettre des messages à Exchange Online ? Cela inclut aussi les chemins hybrides, applicatifs et de secours.
2. **Configurer un connecteur partenaire restrictif.** Utiliser `SenderDomains *` et limiter la remise à un certificat (de préférence) ou à des plages d’adresses IP sources maintenues. Un connecteur de type `OnPremises` ou « Votre organisation » n’impose pas cet effet de refus par défaut (voir par exemple : [Routage du courrier entre Apache James et Exchange Online](/blog/totemomail-m365)).
3. **Configurer correctement Enhanced Filtering.** Si EOP doit continuer à filtrer, l’IP d’origine et les informations de l’expéditeur doivent être correctement reconstituées. Les bypass SCL-`-1` généralisés doivent être examinés avec attention.
4. **Désactiver Direct Send s’il n’est pas utilisé.** Vérifier au préalable, avec Message Trace ou les rapports disponibles, si des scanners ou applications en dépendent.
5. **Ne pas basculer aveuglément.** Tester puis surveiller les plages IP de la passerelle, les changements de certificats, le flux de messagerie hybride ainsi que les chemins particuliers `onmicrosoft.com`, Teams et autres.

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
| `-SenderDomains *` | Le connecteur s’applique aux e-mails de tous les domaines expéditeurs |
| `-RestrictDomainsToIPAddresses $true` | Active le blocage : les e-mails des domaines indiqués ne sont plus acceptés que depuis les adresses définies dans `-SenderIpAddresses` |
| `-SenderIpAddresses` | Les adresses ou plages IP sources autorisées de la passerelle en amont |
| `-RequireTls $true` | Exige le chiffrement TLS pour les connexions via ce connecteur |

</details>

Lorsque cela est possible, la liaison par certificat est préférable à une liste d’autorisation IP. Les modifications doivent d’abord être effectuées dans un test contrôlé, car une liste d’autorisation erronée transforme très vite l’entrée latérale ouverte en panne totale de messagerie.

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
| `-SmtpServer` | Hôte cible : le point de terminaison Exchange Online public du tenant, volontairement en contournant le MX |
| `-To` | Adresse du destinataire dans le tenant à tester |
| `-From` | Adresse d’expéditeur externe quelconque ; c’est précisément ce que l’entrée latérale ne devrait plus accepter |
| `-Subject` | Objet du message, pour le retrouver dans Message Trace |
| `-Body` | Corps du message |

</details>

Avec un connecteur partenaire correctement restreint, il faut s’attendre à un rejet SMTP tel que `5.7.51 TenantInboundAttribution; Rejecting`. Une règle de transport alternative peut d’abord accepter le message, puis le déplacer en quarantaine ; il faut donc contrôler non seulement la réponse SMTP, mais aussi Message Trace, la quarantaine et la boîte aux lettres. `Send-MailMessage` (obsolète) ne sert ici qu’à illustrer simplement le principe. Tout outil de test SMTP contrôlé remplit le même objectif.

## Un test utile avec une étiquette trompeuse

« Ghost Sender » n’est pas un nouvel exploit SMTP. C’est un nom percutant pour une entrée latérale ouverte, dont la sécurisation est documentée depuis longtemps par Microsoft et que l’administrateur a laissée ouverte.

L’ironie est qu’InfoGuard qualifie lui-même le problème, dans son propre article, de « widespread and systematic misconfiguration » et conclut par « Ghost-Sender is a misconfiguration ». Le Microsoft Security Response Center n’a pas non plus initialement considéré le signalement comme une faille de sécurité. Les faits sont donc bien présents dans l’article : seuls le titre, l’e-mail de test et l’étiquette « Vulnerability » suggèrent malheureusement une interprétation plus dramatique.

La partie utile de la publication est son rôle d’alerte : de nombreuses entreprises n’ont apparemment pas correctement verrouillé leur flux de messagerie. La partie problématique est l’affirmation selon laquelle Exchange Online présenterait pour cette raison une faille de sécurité universelle. Non : Exchange Online se comporte d’abord ici comme un MTA. Ce qui le rend non sécurisé, c’est la limite de confiance qui n’a pas été entièrement configurée.

Microsoft doit-il vraiment tout faire à la place de l’administrateur ? Non. Mais il faut manifestement rappeler sans cesse que le routage DNS ne remplace pas le contrôle d’accès.

## Sources

1.  [InfoGuard Labs : Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/) : l’étude initiale, avec son analyse de diffusion et sa propre conclusion : « Ghost-Sender is a misconfiguration ».

2.  [Ghost Sender : Exchange Online Mail Spoofing Tester](https://ghost-sender.com/) : le test en ligne publié par InfoGuard pour vérifier si son propre tenant présente une entrée latérale ouverte.

3.  [MSXFAQ : Exchange Online comme entrée latérale pour la réception de courrier](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) : l’analyse de Frank Carius : pas d’erreur dans Exchange Online, mais une mauvaise configuration de l’administrateur.

4.  [Microsoft : Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865) : Microsoft explique que l’acceptation directe de courrier destiné à des boîtes aux lettres hébergées relève du fonctionnement normal de l’e-mail, et distingue Direct Send.

5.  [Microsoft Learn : Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud) : les instructions officielles, avec l’étape dédiée au connecteur partenaire restrictif pour un MX externe.

6.  [Microsoft Learn : Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : reconstitue la source d’expéditeur d’origine derrière une passerelle ; améliore l’évaluation, mais ne remplace pas le connecteur.

7.  [Heise : Ghost-Sender – Exchange Online laisse passer les e-mails falsifiés sans difficulté](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html) : exemple de couverture dramatique qui généralise certaines mauvaises configurations.

8.  [Crow in the Cloud : Les fantômes que je n’ai pas appelés](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) : analyse pertinente en tant que problème de conception et de configuration, avec mesures de protection.

9.  [RFC 5321 : Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html) : décrit l’enregistrement MX comme un mécanisme permettant d’identifier le système de destination habituel, et non comme un contrôle d’accès.

10.  [RFC 9989 : DMARC](https://www.rfc-editor.org/rfc/rfc9989.html) : précise que le destinataire peut prendre en compte le traitement DMARC publié, mais n’y est pas tenu.

---

## Votre flux de messagerie est-il sécurisé ?

Vous ne savez pas si votre tenant Exchange Online possède lui aussi une entrée latérale ouverte ? **adeptio** examine l’ensemble de votre flux de messagerie : des enregistrements MX, connecteurs et passerelles tierces jusqu’à EOP, SPF, DKIM, DMARC et Direct Send. De manière pratique, indépendante et avec des recommandations concrètes.

Pour faire vérifier ou sécuriser correctement votre flux de messagerie, vous pouvez volontiers convenir d’un entretien de conseil sans engagement :

**[Réserver un entretien de conseil avec adeptio](/kontakt)**  
[adeptio.ch](https://adeptio.ch/)
