---
title: "Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu"
navTitle: "Ghost Sender"
description: "La remise directe à Exchange Online contourne une passerelle en amont si le tenant ne la bloque pas explicitement. Le risque est réel, mais sa cause est une configuration de flux de messagerie incomplète."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lecture"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/fr/blog/ghost-sender-dans-exchange-online-un-enregistrement-mx-n-est-pas-un-pare-feu"
translationId: article-d8dc8d1da6379d67
translationReview: automatic
translationSourceHash: fc228adeba2a4ea46f6b36d20946d0aeb5c30f485b32da965e52168d2806a689
translatedAt: 2026-07-29T12:29:38.935Z
---

# Ghost Sender dans Exchange Online : un enregistrement MX n’est pas un pare-feu

![Un administrateur fantôme tient ouverte, dans le centre de données, la porte à côté du portail de sécurité, tandis que des e-mails parviennent directement à la boîte aux lettres en contournant le filtre.](../images/ghost-admin.png)

La possibilité d’attaque décrite par InfoGuard Labs sous le nom de « Ghost Sender » est réelle : un attaquant peut contourner une passerelle de messagerie en amont et remettre des messages directement à Exchange Online. La condition est toutefois que le tenant continue d’accepter cette voie directe. Il ne s’agit pas d’une vulnérabilité universelle d’Exchange Online, mais d’une topologie de flux de messagerie insuffisamment sécurisée.

Un agent de transfert de courrier qui sert les boîtes aux lettres d’un domaine accepte par principe des connexions SMTP depuis Internet. L’enregistrement MX indique aux expéditeurs légitimes le chemin de remise souhaité. Il n’est ni une règle de pare-feu ni une liste de contrôle d’accès et n’empêche personne de s’adresser directement à un point de terminaison Exchange Online connu.

## Ce que « Ghost Sender » démontre réellement

Le scénario décrit par [InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) est le suivant :

1. Une organisation héberge ses boîtes aux lettres dans Exchange Online.
2. L’enregistrement MX public pointe vers une passerelle de messagerie sécurisée en amont.
3. Le point de terminaison Exchange Online sous `*.mail.protection.outlook.com` reste directement accessible depuis Internet.
4. L’administrateur n’a pas restreint Exchange Online afin que seule la passerelle en amont puisse y remettre des messages.
5. Un attaquant ignore l’enregistrement MX et remet son message directement à Exchange Online.

Le chemin prévu est donc :

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Mais ce chemin est resté ouvert :

```text
Angreifer -> Exchange Online -> Postfach
```

Il s’agit d’une erreur de configuration à prendre au sérieux. Le filtre en amont peut être contourné par cette voie ; l’usurpation d’expéditeur, le phishing et la fraude au président s’en trouvent considérablement facilités. InfoGuard mérite d’être salué pour avoir rendu le problème visible, étudié son ampleur et publié un test simple d’utilisation.

Mais où se trouve exactement l’erreur du produit ?

La dramatisation médiatique aide également peu à comprendre la situation. [Heise titre qu’Exchange Online laisse passer des e-mails falsifiés « sans broncher »](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), alors que seules certaines configurations tierces et hybrides insuffisamment durcies sont concernées. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) le formule bien plus précisément : il ne s’agit pas d’une faille de sécurité au sens strict, mais d’un problème de conception et de configuration.

## « An MTA is doing MTA-Things »

Chaque tenant Exchange Online possède un point de terminaison SMTP public. Ce point de terminaison n’est pas secret et ne doit pas l’être. Microsoft explique lui-même qu’Exchange Online accepte par défaut les messages directement adressés aux boîtes aux lettres qui y sont hébergées : [c’est tout simplement le fonctionnement de l’e-mail](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

[SMTP lui-même décrit également l’enregistrement MX comme un mécanisme permettant de déterminer le système de destination habituel](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Il n’en découle aucune obligation pour le serveur cible de rejeter les connexions passant par tout autre hôte accessible. Un attaquant n’est pas tenu de suivre le chemin indiqué. Si un autre MTA est accessible, connaît le domaine destinataire et accepte le message, il sera essayé, à l’instar des spammeurs qui tentent depuis des décennies de contacter des systèmes MX de secours moins bien protégés.

Lorsqu’on place un filtre tiers en amont, on modifie la topologie standard. « Exchange Online est ma passerelle de messagerie Internet » devient « seule ma passerelle tierce peut transmettre du courrier Internet à Exchange Online ». Cette nouvelle `Trust-Border` ne résulte pas d’une entrée DNS. Elle doit être explicitement imposée sur le système récepteur.

C’est précisément ce que documente Microsoft : avec un MX externe, il convient de créer un connecteur entrant de type `Partner` qui n’accepte, pour `SenderDomains *`, que le certificat ou les adresses IP source du service en amont. Les messages remis directement en contournant la passerelle sont alors rejetés. C’est écrit noir sur blanc dans le guide Microsoft [« Manage mail flow using a third-party cloud service with Exchange Online »](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius décrit également en détail cette « entrée latérale » dans la [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM et DMARC ne sont pas des videurs

InfoGuard présente des messages pour lesquels SPF, DKIM et DMARC échouent, mais qui arrivent tout de même dans la boîte aux lettres. Cela paraît spectaculaire, mais il ne s’agit pas d’un « contournement » cryptographique de ces mécanismes. Les e-mails ne passent justement pas les contrôles avec succès. Ils fournissent `fail`. L’élément déterminant est l’action locale que le système destinataire déduit de ce résultat.

SPF vérifie si un système est autorisé à envoyer pour l’expéditeur d’enveloppe. DKIM vérifie une signature. DMARC relie ces résultats au domaine d’expéditeur visible et publie un traitement souhaité. Même la norme DMARC actuelle, [RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1), précise expressément que le destinataire peut prendre en compte ce traitement souhaité, mais n’y est pas obligé. DMARC est un signal important, mais pas un contrôle d’accès réseau.

Avec une passerelle en amont, Exchange Online voit en outre d’abord l’adresse IP de cette passerelle et non celle de l’expéditeur initial. C’est à cela que sert [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : il reconstruit la source d’origine et améliore les évaluations SPF, DKIM, DMARC, anti-usurpation et anti-phishing. Mais Enhanced Filtering n’est pas non plus une serrure. Il ne remplace pas le connecteur partenaire restrictif.

La mauvaise configuration devient particulièrement évidente lorsqu’un administrateur affaiblit ou élimine le contrôle EOP via un contournement SCL, au motif que le produit en amont est déjà censé filtrer, tout en laissant ouverte la remise directe depuis Internet. Dans ce cas, aucun mécanisme de protection n’a été « contourné » : il a délibérément prévu qu’une des deux entrées ne dispose plus de protection efficace.

On peut tout à fait critiquer Microsoft lorsqu’un message présentant un échec d’authentification clairement visible arrive dans la boîte de réception sans avertissement. On peut critiquer la sémantique des types de connecteurs, la documentation et l’absence d’avertissements dans Configuration Analyzer. Tous ces points sont légitimes. L’existence d’un point de terminaison SMTP publiquement accessible n’est toutefois pas une faille de sécurité.

## « Direct Send » n’est pas synonyme de « remise directe »

Deux notions sont confondues dans la discussion :

- **Direct Send** désigne chez Microsoft des messages anonymes dont l’expéditeur d’enveloppe (`5321.MailFrom`) utilise un domaine accepté propre au tenant.
- **La remise directe à Exchange Online** désigne de manière générale un message SMTP qui ignore le MX tiers publié et est remis directement au point de terminaison Exchange. L’expéditeur peut également utiliser n’importe quel domaine externe.

Le paramètre

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

est pertinent lorsque Direct Send n’est pas nécessaire. Il empêche l’usurpation d’un domaine interne par cette voie. Mais il ne ferme pas toute l’entrée latérale pour des expéditeurs externes quelconques. Microsoft décrit le périmètre exact dans la [documentation de cmdlet relative à `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Pour empêcher totalement « Ghost Sender », il faut toujours une restriction d’accès via un connecteur partenaire ou une règle de flux de messagerie appropriée.

## Microsoft doit-il vraiment tout faire à la place de l’administrateur ?

Non. Celui qui ajoute un filtre de messagerie supplémentaire à une chaîne de transport de production assume la responsabilité de cette chaîne de transport.

Le fournisseur ne peut pas deviner de manière fiable si, outre le MX externe, des scanners, appareils multifonctions, services SaaS, serveurs hybrides, relais partenaires ou d’autres systèmes légitimes doivent envoyer directement à Exchange Online. Un blocage automatique fondé sur « le MX pointe ailleurs, donc bloquons tout le reste » interromprait des flux de messagerie souhaités dans de nombreux environnements réels. L’administrateur doit donc définir explicitement la frontière de confiance souhaitée.

Microsoft peut néanmoins faciliter la tâche des responsables. Un bon Configuration Analyzer devrait détecter un MX externe sans connecteur partenaire restrictif et afficher un avertissement clair. L’assistant de configuration pourrait expliquer qu’un connecteur de type « Votre organisation » identifie certes les connexions appropriées, mais ne rejette pas automatiquement les connexions inappropriées. Des options sécurisées par défaut et de meilleurs rapports d’exploitation seraient également bienvenus.

Ce serait un durcissement produit judicieux. Cela ne change toutefois rien à l’analyse technique : une topologie spéciale non sécurisée reste une configuration non sécurisée et ne devient pas un zero-day du seul fait de sa large diffusion.

## Comment fermer l’entrée latérale

Pour les environnements dotés d’un filtre en amont, les points suivants doivent au minimum figurer sur la liste de contrôle :

1. **Documenter intégralement le flux de messagerie.** Quels systèmes sont réellement autorisés à remettre des messages à Exchange Online ? Cela inclut les chemins hybrides, applicatifs et de secours.
2. **Configurer un connecteur partenaire restrictif.** Utiliser `SenderDomains *` et limiter la remise à un certificat (de préférence) ou à des plages d’adresses IP source maintenues. Un connecteur de type `OnPremises` ou « Votre organisation » n’impose pas cet effet de refus par défaut (voir aussi, par exemple : [Routage du courrier entre Apache James et Exchange Online](/blog/totemomail-m365)).
3. **Configurer correctement Enhanced Filtering.** Si EOP doit continuer à filtrer, l’adresse IP d’origine et les informations d’expéditeur doivent être correctement reconstruites. Les contournements généraux de SCL-`-1` doivent être examinés avec attention.
4. **Désactiver Direct Send s’il n’est pas utilisé.** Vérifier auparavant, à l’aide de Message Trace ou des rapports disponibles, si des scanners ou des applications en dépendent.
5. **Ne pas basculer à l’aveugle.** Tester puis surveiller les plages IP de la passerelle, les changements de certificat, le flux de messagerie hybride ainsi que les chemins spéciaux `onmicrosoft.com`, Teams et autres.

Un exemple simplifié de la variante basée sur les adresses IP :

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <plages-IP-de-la-passerelle> `
  -RequireTls $true
```

Lorsque cela est possible, la liaison par certificat est préférable à une liste d’autorisation IP. Les modifications doivent d’abord être effectuées dans un test contrôlé, car une liste d’autorisation erronée transforme très vite l’entrée latérale ouverte en panne complète de messagerie.

## Le test simple à réaliser soi-même

Le test présenté par InfoGuard (et MSXFAQ) est utile :

```powershell
Send-MailMessage `
  -SmtpServer <nom-du-tenant>.mail.protection.outlook.com `
  -To admin@<domaine-du-tenant> `
  -From noreply@example.com `
  -Subject "Entrée latérale EXO" `
  -Body "E-mail de test directement au tenant"
```

Avec un connecteur partenaire correctement restreint, il faut s’attendre à un rejet SMTP tel que `5.7.51 TenantInboundAttribution; Rejecting`. Une règle de transport alternative peut d’abord accepter le message puis le placer en quarantaine ; il convient donc de contrôler, outre la réponse SMTP, Message Trace, la quarantaine et la boîte aux lettres. `Send-MailMessage` (obsolète) ne sert ici qu’à une illustration facile à comprendre. Tout outil de test SMTP contrôlé remplit le même objectif.

## Un test utile avec une étiquette trompeuse

« Ghost Sender » n’est pas un nouvel exploit SMTP. C’est un nom accrocheur pour une entrée latérale ouverte, dont Microsoft documente la sécurisation depuis longtemps et que l’administrateur a laissée ouverte.

L’ironie est qu’InfoGuard qualifie lui-même le problème, dans son propre article, de « widespread and systematic misconfiguration » et conclut par la phrase « Ghost-Sender is a misconfiguration ». Le Security Response Center de Microsoft n’a pas non plus initialement considéré le signalement comme une faille de sécurité. Les faits sont donc bel et bien présents dans l’article : seuls le titre, l’e-mail de test et l’image de marque « Vulnerability » racontent malheureusement une histoire plus dramatique.

La partie utile de la publication est l’avertissement : de nombreuses entreprises n’ont apparemment pas correctement verrouillé leur flux de messagerie. La partie problématique est l’affirmation selon laquelle Exchange Online présenterait pour cette raison une faille de sécurité universelle. Non : Exchange Online se comporte ici d’abord comme un MTA. Il devient non sécurisé en raison d’une frontière de confiance incomplètement configurée.

Microsoft doit-il vraiment tout faire à la place de l’administrateur ? Non. Mais il faut manifestement rappeler encore et encore que le routage DNS ne remplace pas le contrôle d’accès.

## Sources

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/) : L’étude originale, avec l’analyse de diffusion et sa propre conclusion : « Ghost-Sender is a misconfiguration ».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/) : Le test en ligne publié par InfoGuard pour vérifier si son propre tenant possède une entrée latérale ouverte.

3.  [MSXFAQ: Exchange Online comme entrée latérale pour la réception d’e-mails](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) : L’analyse de Frank Carius : pas une erreur dans Exchange Online, mais une mauvaise configuration de l’administrateur.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865) : Microsoft explique que l’acceptation directe des e-mails vers des boîtes aux lettres hébergées correspond au fonctionnement de l’e-mail, et distingue ce cas de Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud) : Le guide officiel, avec sa propre étape consacrée au connecteur partenaire restrictif en cas de MX externe.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) : Reconstruit la source de l’expéditeur d’origine derrière une passerelle ; améliore l’évaluation, mais ne remplace pas le connecteur.

7.  [Heise: Ghost-Sender – Exchange Online laisse passer sans broncher des e-mails falsifiés](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html) : Exemple de couverture sensationnaliste qui généralise certaines mauvaises configurations.

8.  [Crow in the Cloud: Les fantômes que je n’ai pas appelés](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) : Analyse pertinente du problème de conception et de configuration, avec des mesures de protection.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html) : Décrit l’enregistrement MX comme un mécanisme permettant de déterminer le système de destination habituel, et non comme un contrôle d’accès.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html) : Précise que le destinataire peut prendre en compte le traitement DMARC publié, mais n’y est pas obligé.

---

## Votre flux de messagerie est-il sécurisé ?

Vous ne savez pas si votre tenant Exchange Online possède lui aussi une entrée latérale ouverte ? **adeptio** vérifie l’ensemble de votre flux de messagerie : des enregistrements MX, connecteurs et passerelles tierces jusqu’à EOP, SPF, DKIM, DMARC et Direct Send. De manière pratique, indépendante et avec des recommandations concrètes.

Pour faire vérifier ou sécuriser correctement votre flux de messagerie, vous pouvez volontiers convenir d’un entretien de conseil sans engagement :

**[Réserver un entretien de conseil avec adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
