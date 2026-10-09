---
title: "Authentification des e-mails : SPF, DKIM et DMARC"
blatt: "mail-auth"
description: "SPF, DKIM et DMARC expliqués sur le plan technique : identités, évaluation DNS, signatures, alignement, politiques, rapports, redirections, limites de confiance et exploitation pour les administrateurs de messagerie."
fakten:
  - label: Objectif SPF
    wert: l’IP autorise RFC5321.MailFrom ou HELO
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-2
  - label: Publication SPF
    wert: un enregistrement TXT avec v=spf1
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-3
  - label: Budget DNS SPF
    wert: au plus 10 termes déclenchant une recherche
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4
  - label: Objectif DKIM
    wert: signature de domaine sur des en-têtes sélectionnés et le corps
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3
  - label: Clé DKIM
    wert: selector._domainkey.signing-domain
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3.6.2.1
  - label: Cryptographie DKIM
    wert: RSA-SHA256 ou Ed25519-SHA256
    href: https://datatracker.ietf.org/doc/html/rfc8463#section-3
  - label: Norme DMARC
    wert: RFC 9989 ; reporting dans les RFC 9990 et 9991
    href: https://datatracker.ietf.org/doc/html/rfc9989
  - label: Identité DMARC
    wert: une Author Domain issue d’un unique champ From
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.2
  - label: Passage DMARC
    wert: SPF ou DKIM réussit et est aligné
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Alignement
    wert: relaxed par défaut ; strict facultatif
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Politiques
    wert: none · quarantine · reject
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.7
  - label: Preuve opérationnelle
    wert: Authentication-Results plus rapports agrégés
    href: https://datatracker.ietf.org/doc/html/rfc8601
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 1247c1771c81b476bf23da2eeee6feb35a3d16c7146e2c61d6c0fa5625c55bbd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:27:28.193Z
translationReview: automatic
---

# Authentification des e-mails : SPF, DKIM et DMARC

SPF, DKIM et DMARC répondent à trois questions différentes concernant le domaine d’expéditeur utilisé. SPF vérifie l’IP émettrice, DKIM une signature cryptographique et DMARC le lien entre ces deux résultats et le domaine From visible. Aucun de ces mécanismes n’authentifie une personne ni ne prouve qu’un message est inoffensif. Un passage DMARC signifie uniquement que l’utilisation de l’Author Domain a été autorisée selon les règles publiées ([RFC 7208, section 1](https://datatracker.ietf.org/doc/html/rfc7208#section-1), [RFC 6376, section 1](https://datatracker.ietf.org/doc/html/rfc6376#section-1), [RFC 9989, section 1](https://datatracker.ietf.org/doc/html/rfc9989#section-1)).

Pour les administrateurs de messagerie, le système est avant tout une chaîne de responsabilités. Le service sortant doit générer des domaines Envelope appropriés et des signatures DKIM. Le [DNS](/kb/dns) autoritatif doit fournir correctement et à temps la politique SPF, les clés DKIM et la politique DMARC. Le serveur de périphérie [SMTP](/kb/smtp) récepteur requiert l’IP client d’origine, un résolveur, une vérification cryptographique et une limite de confiance définie pour les résultats. Un collecteur de rapports doit traiter de manière sécurisée du XML non fiable, détecter les doublons et rendre les données exploitables dans le temps. Un `dmarc=pass` n’est que le résultat final de cette chaîne distribuée.

L’explication suit les identités d’un e-mail : expéditeur Envelope, domaine From visible et signature DKIM. SPF et DKIM sont d’abord expliqués séparément, puis DMARC relie leurs résultats via l’alignement, la politique et le reporting.

## Identités et limites de confiance

Un message comporte plusieurs notions d’expéditeur qui ne sont pas interchangeables. Le **RFC5321.MailFrom** est le chemin de retour de l’enveloppe SMTP et le principal objet SPF. Avec un chemin de retour vide, comme prévu pour les rapports de livraison, SPF dérive l’identité de HELO/EHLO. Le **RFC5322.From** figure dans l’en-tête du message, est affiché par le client de messagerie et fournit à DMARC l’Author Domain. Une signature DKIM indique avec `d=` son Signing Domain et avec `s=` le sélecteur de la clé publique. SMTP AUTH authentifie quant à lui un client auprès d’un service de soumission, mais n’est ni SPF, ni DKIM, ni DMARC ([RFC 5321, sections 3.3 et 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321#section-3.3), [RFC 5322, section 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2), [RFC 7208, sections 2.3 et 2.4](https://datatracker.ietf.org/doc/html/rfc7208#section-2.3), [RFC 6376, section 3.5](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

| Identité | Source | Vérificateur | Affirmation principale |
|---|---|---|---|
| adresse IP de connexion | connexion TCP au MTA récepteur | SPF | cet hôte est ou n’est pas autorisé pour le domaine SMTP vérifié |
| domaine HELO/EHLO | dialogue SMTP | SPF | identité de domaine du client SMTP |
| domaine RFC5321.MailFrom | enveloppe SMTP | SPF et DMARC | domaine Bounce ou Return-Path |
| DKIM `d=` et `s=` | `DKIM-Signature` | DKIM et DMARC | Signing Domain et sélecteur de clé |
| domaine RFC5322.From | en-tête de message visible | DMARC | Author Domain avec laquelle l’alignement doit être établi |
| `authserv-id` | `Authentication-Results` | consommateurs internes | quel service de vérification de confiance a généré le résultat |

Cette séparation constitue une limite de sécurité. Un attaquant peut réussir entièrement SPF et DKIM avec son propre domaine tout en mentionnant une marque tierce dans le nom d’affichage. DMARC limite l’utilisation non autorisée du domaine dans le `From:` visible, mais pas les domaines ressemblants, la fraude au nom d’affichage, les expéditeurs légitimes compromis ni les contenus malveillants ([RFC 7208, section 11.2](https://datatracker.ietf.org/doc/html/rfc7208#section-11.2), [RFC 9989, sections 2.2 et 11.4](https://datatracker.ietf.org/doc/html/rfc9989#section-11.4)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-mail-auth.svg?v=20260813" title="Interaktive Infografik: Identitäten, Alignment, DMARC-Entscheidung, Reporting und indirekte Nachrichtenflüsse" loading="lazy">
  <a href="/images/kb-interaktiv-mail-auth.svg?v=20260813">Ouvrir l’infographie sur SPF, DKIM et DMARC</a>
</iframe>

## SPF : autorisation de l’IP de connexion

Le Sender Policy Framework est une autorisation basée sur le DNS pour le domaine figurant dans `MAIL FROM` ou HELO. Le destinataire évalue l’IP client, le domaine vérifié, l’identité d’expéditeur et le nom d’hôte local avec la fonction `check_host()` définie dans la RFC 7208. L’enregistrement est publié comme ressource TXT directement sur le domaine concerné et commence par `v=spf1`; le type DNS RR SPF abandonné n’est pas utilisé ([RFC 7208, sections 3.1 et 4.1](https://datatracker.ietf.org/doc/html/rfc7208#section-3.1)).

Un enregistrement tel que `v=spf1 ip4:192.0.2.0/24 include:_spf.sender.example -all` est évalué de gauche à droite. Un mécanisme correspondant termine le traitement avec son qualificateur. `+` signifie `pass` et est la valeur par défaut, `-` signifie `fail`, `~` signifie `softfail`, `?` signifie `neutral`. `all`, `ip4` et `ip6` ne nécessitent aucune recherche DNS supplémentaire pendant l’évaluation normale ; `include`, `a`, `mx`, `ptr`, `exists` et `redirect` en nécessitent ([RFC 7208, sections 4.6.1 à 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6)).

| Résultat | Signification protocolaire | Question d’administration |
|---|---|---|
| `pass` | l’IP est autorisée pour cette identité SPF | Ce domaine est-il également aligné avec le RFC5322.From ? |
| `fail` | la politique publiée n’autorise pas l’IP | mauvaise source, utilisation frauduleuse du domaine ou politique obsolète ? |
| `softfail` | affirmation négative faible du domaine | sert-il encore à une phase de transition contrôlée ou masque-t-il une dérive ? |
| `neutral` | aucune affirmation d’autorisation | manque-t-il un mécanisme final ou la neutralité est-elle voulue ? |
| `none` | aucune politique SPF applicable | le bon domaine MailFrom/HELO a-t-il été vérifié ? |
| `temperror` | erreur temporaire, généralement DNS | vérifier le résolveur, le délai d’attente et l’accessibilité des serveurs faisant autorité |
| `permerror` | enregistrement ou évaluation durablement invalide | vérifier la syntaxe, les enregistrements SPF multiples, la récursion et le budget de recherches |

### Budget de recherches et dépendances

SPF limite à dix termes, sur l’ensemble de l’évaluation récursive, la somme de `include`, `a`, `mx`, `ptr`, `exists` et `redirect`. Un dépassement doit produire `permerror`. Des limites d’adresses supplémentaires s’appliquent à `mx` et `ptr` ; plus de deux réponses vides ou résultats NXDOMAIN, appelés Void Lookups, doivent également conduire à `permerror` ([RFC 7208, section 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4)).

Le budget est une limite d’exécution et non un simple contrôle de caractères. Un seul `include` peut introduire d’autres inclusions, résolutions MX et domaines de panne. `include` délègue uniquement la question de savoir si l’hôte actuel y obtient un `pass` ; `redirect` reprend l’intégralité de la politique d’un autre domaine après l’échec de la vérification des mécanismes. La RFC 7208 recommande `include` pour franchir les limites administratives et `redirect` plutôt pour centraliser les domaines gérés de manière homogène ([RFC 7208, sections 5.2 et 6.1](https://datatracker.ietf.org/doc/html/rfc7208#section-5.2)). L’inventaire d’exploitation doit donc inclure le propriétaire, l’objectif, le processus de modification et le budget maximal mesuré de chaque inclusion tierce.

### Redirection et domaine SPF

Un redirecteur classique se connecte depuis sa propre IP au destinataire suivant, mais conserve le `MAIL FROM` d’origine. Une IP est alors vérifiée par rapport à la politique SPF d’un domaine tiers et SPF peut échouer bien que la soumission initiale ait été légitime. La RFC 7208 décrit comme contre-mesure la réécriture du Reverse Path vers un domaine de l’intermédiaire ; les listes de diffusion le font souvent de toute façon ([RFC 7208, annexe D.2](https://datatracker.ietf.org/doc/html/rfc7208#appendix-D.2)). Cela rétablit le passage SPF pour le nouveau domaine Envelope, mais ne produit un passage DMARC que si ce nouveau domaine est aligné avec le `From:` visible. Pour les chemins indirects, une signature DKIM préservée et alignée est donc particulièrement importante.

## DKIM : signature d’un domaine

DomainKeys Identified Mail ajoute un champ d’en-tête `DKIM-Signature`. Le signataire canonise les en-têtes sélectionnés et le corps, calcule le hachage du corps `bh=`, signe les données définies et publie la clé sous `s=._domainkey.d=`. Le vérificateur reconstruit les mêmes données, interroge la clé DNS TXT et vérifie la signature ainsi que le hachage du corps. Un passage prouve que les éléments signés n’ont pas été modifiés depuis la signature d’une manière détectable et que le signataire contrôlait la clé privée de la Signing Domain. DKIM ne prouve ni une personne physique ni la véracité du contenu ([RFC 6376, sections 3.5 à 3.8](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

Les balises importantes sont `a=` pour l’algorithme, `c=` pour la canonisation des en-têtes et du corps, `d=` pour la Signing Domain, `s=` pour le sélecteur, `h=` pour les noms d’en-têtes signés, `bh=` pour le hachage du corps et `b=` pour la signature. `t=` et `x=` peuvent transporter l’instant de signature et d’expiration, mais ne constituent pas une protection fiable contre les relectures. Le `l=` facultatif limite la zone de corps signée et peut ainsi permettre l’ajout de contenu non signé ; la RFC 6376 décrit explicitement cette surface d’abus ([RFC 6376, sections 3.5 et 8.2](https://datatracker.ietf.org/doc/html/rfc6376#section-8.2)).

La canonisation `simple` ne tolère pratiquement aucune modification. `relaxed` normalise notamment certaines représentations et les espaces blancs, afin que les modifications habituelles du transport ne rompent pas inutilement une signature. Les en-têtes et le corps peuvent utiliser des procédés différents ; en l’absence de `c=`, `simple/simple` s’applique. La canonisation ne modifie pas le message transmis, mais uniquement sa forme d’entrée pour la signature ou la vérification ([RFC 6376, section 3.4](https://datatracker.ietf.org/doc/html/rfc6376#section-3.4)).

### Clés, algorithmes et rotation

La RFC 8301 exige au moins 1024 bits pour RSA, recommande au moins 2048 bits aux signataires et qualifie RSA-SHA1 d’historique. La RFC 8463 ajoute Ed25519-SHA256 et autorise, pour assurer la compatibilité pendant la transition, des signatures parallèles avec différents sélecteurs ([RFC 8301, sections 3.1 et 3.2](https://datatracker.ietf.org/doc/html/rfc8301#section-3), [RFC 8463, sections 5 et 6](https://datatracker.ietf.org/doc/html/rfc8463#section-5)). Le choix de l’algorithme reste une décision d’interopérabilité : une obligation normalisée du côté vérificateur ne garantit pas automatiquement que chaque plateforme de réception réelle l’implémente sans erreur.

Les sélecteurs séparent le renouvellement des clés du domaine. Pour une rotation, la nouvelle clé publique est d’abord publiée, puis les messages sont signés avec la nouvelle clé privée ; l’ancienne clé DNS n’est retirée que lorsque les anciens messages n’ont plus besoin d’être régulièrement vérifiés. La RFC 6376 déconseille de réutiliser un sélecteur avec une nouvelle clé, car les anciens échecs de signature ne pourraient alors plus être distingués des falsifications. Un `p=` vide dans l’enregistrement de clé révoque la clé ([RFC 6376, sections 3.1 et 6.1.2](https://datatracker.ietf.org/doc/html/rfc6376#section-3.1)).

La clé privée ne doit figurer ni dans le DNS ni dans des dépôts de configuration généraux. La conception opérationnelle doit définir le propriétaire de la clé, sa génération, son stockage protégé, l’accès du signataire, la rotation, la révocation d’urgence, la décision de sauvegarde et la piste d’audit. La RFC 6376 exige de la prudence dans la protection des clés privées et cite le stockage chiffré ainsi que le matériel cryptographique comme mesures de protection possibles. Plusieurs plateformes d’envoi doivent disposer de sélecteurs distincts afin qu’une plateforme compromise puisse être isolée sans renouvellement global des clés. Cette architecture suit la séparation administrative de l’espace de noms des sélecteurs prévue par DKIM ([RFC 6376, sections 3.1 et 8.3 ainsi que l’annexe C](https://datatracker.ietf.org/doc/html/rfc6376#section-8.3)).

SPF peut échouer après une redirection bien que le message soit resté inchangé. DKIM peut au contraire survivre à une redirection, mais être rompu par un pied de page modifié. DMARC relie donc les deux procédés au moyen de l’alignement.

## DMARC : alignement, politique et évaluation

DMARC requiert un unique champ RFC5322.From correctement formé et en extrait exactement une **Author Domain**. Il considère le domaine authentifié par SPF et tous les domaines de signature DKIM vérifiés avec succès. Il y a passage DMARC lorsqu’au moins un résultat SPF ou DKIM est `pass` et que son domaine est aligné avec l’Author Domain. Les deux mécanismes ne doivent donc pas réussir simultanément ; en exploitation, les deux sont souhaitables, car ils peuvent échouer sur différents flux indirects ([RFC 9989, sections 4.2 à 4.4 et 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-4.2)).

En **strict alignment**, les domaines doivent être identiques. En **relaxed alignment**, ils doivent avoir la même Organizational Domain. `adkim=s` ou `aspf=s` impose strict ; en l’absence de ces balises, relaxed s’applique. Selon la RFC 9989, la détermination de l’Organizational Domain se fait par un DNS Tree Walk limité et non plus uniquement à l’aide d’une Public Suffix List. Le parcours interroge au plus huit niveaux de noms et tient compte de `psd=y` ou `psd=n` ([RFC 9989, sections 4.4 et 4.10](https://datatracker.ietf.org/doc/html/rfc9989#section-4.10)). Cette modification est pertinente avec des vérificateurs anciens et nouveaux mixtes ; la RFC 9989 signale elle-même que les résultats d’alignement peuvent différer.

### Enregistrement de politique et balises

L’enregistrement DMARC est situé sous `_dmarc.<domain>` et utilise une syntaxe balise/valeur. `v=DMARC1` identifie le format. `p=` décrit le traitement souhaité pour les messages non réussis du domaine de politique : `none`, `quarantine` ou `reject`. `sp=` peut traiter différemment les sous-domaines existants, `np=` les sous-domaines inexistants. `rua=` désigne les destinations des rapports agrégés, `ruf=` les destinations facultatives des rapports d’échec. `t=y` indique le mode test défini dans la RFC 9989. Les balises inconnues sont ignorées ([RFC 9989, sections 4.5 à 4.8](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)).

Le déploiement progressif antérieur au moyen de `pct=` ne fait plus partie du protocole selon la RFC 9989. L’expérience opérationnelle a montré une application incohérente ; `t=` ne remplace que l’ancien effet particulier de `pct=0`, et non des paliers de pourcentage arbitraires. Une introduction progressive doit donc être planifiée au moyen d’Author Domains distinctes, de politiques de sous-domaines, de flux de messagerie ciblés et d’une évaluation robuste des rapports, et non à travers un supposé régulateur de pourcentage normalisé ([RFC 9989, annexe A.6](https://datatracker.ietf.org/doc/html/rfc9989#appendix-A.6)).

Une politique est une préférence publiée du Domain Owner. Le destinataire peut s’en écarter en raison d’une politique locale, de la réputation, de flux indirects ou d’autres informations, et documente éventuellement les dérogations dans les rapports. `p=reject` ne rend pas automatiquement un message non distribuable au sens SMTP chez chaque destinataire ; cela crée une affirmation claire et automatisable pour le traitement d’un échec DMARC ([RFC 9989, sections 5.3 et 5.4](https://datatracker.ietf.org/doc/html/rfc9989#section-5.3)).

Une politique DMARC ne devrait être renforcée que lorsque tous les chemins d’envoi légitimes sont connus. Les rapports agrégés fournissent les données nécessaires à cet inventaire et indiquent quels systèmes envoient réellement sous un domaine.

## Reporting comme chaîne de données opérationnelle

La RFC 9990 sépare le reporting agrégé de la spécification centrale DMARC. Un rapport regroupe les messages selon l’IP source, la politique évaluée, la disposition, l’alignement et les résultats d’authentification. Le format de données est XML ; le fichier doit être compressé avec GZIP et porte alors l’extension `.xml.gz`. Les enregistrements individuels comprennent notamment `source_ip`, `count`, `header_from`, les résultats SPF et DKIM ainsi que d’éventuels motifs de dérogation à la politique ([RFC 9990, sections 3.1 et 3.4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.1)).

Un collecteur est donc un service d’ingestion pertinent pour la sécurité. Il reçoit des e-mails non sollicités et des pièces jointes, décompresse des données tierces, analyse du XML, déduplique les rapports et agrège les volumes. Les limites de taille, le budget de décompression, un analyseur XML sans entités externes, la protection contre les logiciels malveillants, la quarantaine pour les fichiers défectueux, l’idempotence et la conservation font partie de l’architecture. La RFC 9989 met en garde contre les rapports volontairement malformés et les attaques DoS contre les destinations de reporting ; la RFC 9990 règle les doublons et les destinations externes ([RFC 9989, section 11.2](https://datatracker.ietf.org/doc/html/rfc9989#section-11.2), [RFC 9990, sections 3.5.4 et 4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.5.4)).

Si `rua=` se trouve hors de l’Organizational Domain, le destinataire du rapport doit autoriser cette relation au moyen d’un enregistrement DNS TXT supplémentaire. Ainsi, un domaine ne peut pas inonder des tiers arbitraires de rapports ([RFC 9990, section 4](https://datatracker.ietf.org/doc/html/rfc9990#section-4)). Les Failure Reports de la RFC 9991 peuvent révéler des informations sur des messages individuels. En raison des risques de protection des données, de nombreux opérateurs les restreignent ; les rapports agrégés sont le canal de visibilité recommandé sans contenu d’utilisateur final ([RFC 9991, section 7](https://datatracker.ietf.org/doc/html/rfc9991#section-7)).

Pour l’évaluation administrative, les séries temporelles sont plus importantes qu’une valeur quotidienne isolée : volume par IP source et Author Domain, SPF aligné, DKIM aligné, échec DMARC, `temperror`, `permerror`, sélecteurs inconnus, dérogations à la politique et couverture des rapporteurs. Les rapports sont des observations de destinataires individuels et non une comptabilité exhaustive des envois ; les destinataires ne sont pas tenus de fournir chaque évaluation ou disposition demandée ([RFC 9989, sections 1 et 6](https://datatracker.ietf.org/doc/html/rfc9989#section-6)).

## Faire correctement confiance à Authentication-Results

Le service de vérification récepteur peut consigner les résultats SPF, DKIM et DMARC dans le champ d’en-tête `Authentication-Results`. `authserv-id` identifie le service ou l’Administrative Management Domain ayant effectué la vérification. Ce champ n’a valeur probante qu’à l’intérieur d’une limite de confiance définie. Un expéditeur externe peut lui-même insérer un `Authentication-Results: ... dmarc=pass` convaincant ([RFC 8601, sections 1.5.4 à 1.6](https://datatracker.ietf.org/doc/html/rfc8601#section-1.5.4)).

Le MTA de périphérie doit donc supprimer les instances tierces ou n’autoriser que des producteurs explicitement dignes de confiance. Les consommateurs internes nécessitent une liste de valeurs `authserv-id` autorisées et doivent tenir compte de la position dans la chaîne `Received` de confiance ou dans le modèle interne de provenance des métadonnées. La RFC 8601 met explicitement en garde contre l’activation du champ d’en-tête pour des décisions de filtrage sans MTA de périphérie conforme et vérifié ([RFC 8601, sections 5 et 7.1](https://datatracker.ietf.org/doc/html/rfc8601#section-5)).

ARC selon la RFC 8617 peut transporter dans une chaîne signée les résultats d’authentification et les modifications ultérieures à travers les intermédiaires. ARC est publié comme protocole expérimental et ne fournit pas de racine de confiance globale : le destinataire final décide toujours auxquels des scelleurs ARC il fait confiance. Un statut de chaîne ARC valide est donc un contexte pour la politique locale, non un remplacement de DMARC ni une décision d’acceptation automatique ([RFC 8617, sections 1 et 5](https://datatracker.ietf.org/doc/html/rfc8617#section-1)).

Jusqu’ici, il s’agissait de livraison directe. Les redirections, listes et passerelles modifient toutefois l’adresse IP, l’enveloppe ou le contenu, c’est-à-dire précisément les entrées des trois vérifications.

## Flux indirects et points de rupture typiques

Les redirections, listes de diffusion, passerelles de sécurité et systèmes de tickets modifient différentes parties du message. Une redirection change l’IP de connexion et peut rompre SPF. Une liste de diffusion peut modifier l’objet, les en-têtes de liste, le pied de page du corps ou la structure MIME et ainsi rompre DKIM. Elle peut en outre réécrire l’expéditeur Envelope et le `From:` visible. La RFC 7960 décrit ces problèmes d’interopérabilité et leurs effets secondaires respectifs ; il n’existe aucune correction universelle qui préserve simultanément l’identité, la fonction de liste et la politique existante du destinataire ([RFC 7960, sections 3 et 4](https://datatracker.ietf.org/doc/html/rfc7960#section-3)).

Pour le dépannage, les administrateurs doivent donc comparer l’état **avant et après chaque intermédiaire** : IP client, HELO, MailFrom, RFC5322.From, signatures DKIM existantes, `Authentication-Results`, nouveaux champs `Received` et mutations du contenu. Un résultat final `dmarc=fail` ne montre pas à lui seul quel saut a fait perdre l’identité alignée.

## Architecture technique d’une plateforme de production

L’authentification des e-mails n’est pas une appliance unique, mais un plan de contrôle et de données distribué :

| Composant | Pile technologique | État persistant | Domaine de panne central |
|---|---|---|---|
| DNS autoritatif | ensembles TXT-RR, DNSSEC facultatif, modification basée sur zone ou API | SPF, clés publiques DKIM, politique DMARC | enregistrements obsolètes, Split-Horizon, TTL, délégation défectueuse |
| Signataire sortant | filtre MTA, bibliothèque ou passerelle ; RSA/Ed25519 et SHA-256 | clés privées, configuration du sélecteur, politique de signature | accès aux clés, `d=` incorrect, signature absente sur certains flux |
| Vérificateur entrant | MTA/filtre de périphérie, résolveur récursif, bibliothèque cryptographique, moteur de politique | cache du résolveur, limite de confiance, dérogations locales | IP client incorrecte après proxy, délai DNS, résultats d’authentification manipulés |
| Module de politique DMARC | alignement, DNS Tree Walk, logique de domaine/politique | cache et version de politique | ancien modèle RFC 7489, Organizational Domain incorrecte |
| Générateur de rapports | télémétrie MTA, agrégateur, XML/GZIP, envoi SMTP | fenêtres temporelles, enregistrements, ID de rapport | perte de données, doublons, couverture incomplète des rapporteurs |
| Collecteur de rapports | boîte aux lettres, décompresseur, analyseur XML, base de données, tableau de bord | rapports bruts, normalisation, séries temporelles | attaque sur l’analyseur, déduplication incorrecte, conservation non contrôlée |

Le langage de programmation et le produit sont interchangeables, les objets de protocole et les limites de confiance ne le sont pas. Un MTA peut implémenter la signature et la vérification en C, Rust, Java, Go ou par l’intermédiaire d’un processus de filtrage distinct. Les mêmes questions sont déterminantes en exploitation : d’où provient l’IP client après les équilibreurs de charge ? Quel résolveur et quel cache sont utilisés ? Où se trouve la clé privée ? Quel processus a le droit de signer ? Quels en-têtes sont supprimés avant la limite de confiance ? Comment corréler la version de politique, la réponse DNS, le Message-ID, le Queue-ID et le résultat ? Les RFC définissent la sémantique filaire et d’évaluation, non le modèle de processus concret.

### Processus contrôlé d’introduction et de modification

La RFC 9989 décrit un ordre robuste pour les Domain Owners : publier SPF aligné, configurer DKIM aligné, créer une boîte aux lettres de rapports agrégés, publier d’abord DMARC en mode surveillance avec `p=none`, évaluer les rapports, combler les lacunes et décider seulement ensuite de l’application ([RFC 9989, section 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-5.1)). Il en résulte un processus vérifiable pour la gestion des changements :

1. Inventorier tous les domaines Author, domaines Envelope, noms HELO, produits d’envoi, tenants, relais, redirecteurs et propriétaires de clés.
2. Prouver au moins une identité alignée pour chaque flux de messagerie légitime ; SPF et DKIM ensemble réduisent la dépendance à un seul chemin indirect.
3. Rendre l’acceptation des rapports et leur évaluation sécurisée opérationnelles avant `rua=`.
4. Exploiter `p=none` comme phase de mesure, sans le confondre avec un effet de protection.
5. Classer les sources inconnues : légitimes mais mal configurées, mises hors service, abusives ou altérées par un flux indirect.
6. Introduire l’application par domaine contrôlable, documenter les exceptions et observer au moyen des rapports et de la télémétrie de distribution.
7. Tester régulièrement la rotation des clés, le changement de fournisseur, le rollback DNS, la panne de reporting et un signataire compromis.

Une validation ne devrait pas se limiter à l’acceptation d’un enregistrement TXT syntaxiquement valide. Elle nécessite des messages de test sur chaque chemin d’envoi, la preuve d’en-tête chez le destinataire, des données de rapport de plusieurs domaines destinataires, la mesure du budget SPF, la vérification des TTL de sélecteurs et un rollback qui ne laisse pas l’Author Domain non protégée ou non distribuable.

Ces dépendances imposent un ordre de diagnostic fixe : identifier d’abord les domaines utilisés, puis vérifier le DNS et la signature, et évaluer enfin l’alignement et la politique DMARC.

## Outils de diagnostic

Les requêtes suivantes utilisent `example.ch` et le sélecteur d’exemple `s2026a`. Les noms de production et les messages stockés localement ne doivent être examinés que dans des environnements autorisés.

### Lire SPF, DMARC et DKIM dans le DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail-Authentifizierungs-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type TXT -DnsOnly
Resolve-DnsName _dmarc.example.ch -Type TXT -DnsOnly
Resolve-DnsName s2026a._domainkey.example.ch -Type TXT -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch TXT
dig +noall +answer _dmarc.example.ch TXT
dig +noall +answer s2026a._domainkey.example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) affichent les ensembles TXT-RR, mais pas encore leur évaluation protocolaire complète. Les plusieurs chaînes de caractères d’un même enregistrement TXT doivent être concaténées sans caractères supplémentaires ; plusieurs enregistrements SPF ou DMARC au même nom constituent en revanche une erreur ([RFC 7208, sections 3.2 et 4.5](https://datatracker.ietf.org/doc/html/rfc7208#section-3.2), [RFC 9989, section 4.5](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)). Pour les questions de Split-Horizon ou de DNSSEC, les vues autoritatives et récursives doivent être vérifiées séparément.

### Trouver les résultats de confiance dans l’en-tête du message

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Header-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\message.eml |
  Select-String -Pattern '^(Authentication-Results|DKIM-Signature|Received):' -Context 0,8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grep -E -A8 '^(Authentication-Results|DKIM-Signature|Received):' message.eml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) et [`Select-String`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string), ou [`grep`](https://www.gnu.org/software/grep/manual/grep.html), aident à la première inspection. En raison du repliement des en-têtes et des champs multiples, une recherche textuelle ne remplace pas un analyseur conforme aux RFC. Sont déterminants le `authserv-id` digne de confiance, sa position par rapport à la limite `Received` propre, les domaines réellement vérifiés et la distinction entre résultat brut et alignement.

### Vérifier structurellement un rapport agrégé

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DMARC-XML-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">[xml]$report = Get-Content .\report.xml -Raw
$report.SelectNodes("//*[local-name()='record']").Count
$report.SelectSingleNode("//*[local-name()='org_name']").InnerText</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">xmllint --noout report.xml
xmllint --xpath 'count(//*[local-name()="record"])' report.xml
xmllint --xpath 'string(//*[local-name()="org_name"])' report.xml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) charge ici un fichier XML local déjà décompressé ; [`xmllint`](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) vérifie la structure et les requêtes XPath. Les rapports inconnus ne doivent pas être ouverts de manière interactive avec des outils de bureau privilégiés. Cette vérification ponctuelle ne prouve ni la conformité au schéma ni la sécurité du traitement de masse, de la détection des doublons ou de l’agrégation correcte.

### Associer systématiquement les erreurs à une limite

| Observation | Cause probable | Preuve solide suivante |
|---|---|---|
| `spf=none` | identité incorrecte ou absence d’enregistrement SPF | MailFrom/HELO du résultat SMTP ou d’authentification et TXT au nom exact |
| `spf=permerror` | syntaxe, enregistrements multiples, récursion ou budget DNS | évaluer l’arbre SPF récursif complet et les compteurs Lookup/Void |
| SPF réussit, DMARC échoue | domaine SPF non aligné | comparer RFC5321.MailFrom à RFC5322.From dans le mode d’alignement configuré |
| `dkim=fail (body hash did not verify)` | corps modifié après la signature | comparer le saut de signature, la normalisation MIME, le pied de page/disclaimer et la canonisation |
| `dkim=temperror` | interrogation de clé temporairement échouée | nom du sélecteur, réponse du résolveur, délai d’attente et accessibilité DNS autoritative |
| DKIM réussit, DMARC échoue | seule une signature non alignée réussit | vérifier tous les domaines `d=` individuellement par rapport à l’Author Domain |
| les destinataires signalent des résultats DMARC différents | chemins, caches DNS, modèles de vérificateur ou mutations différents | corréler le même Message-ID/la même signature à un en-tête complet et à l’instant DNS pour chacun |
| IP source inconnue dans les rapports agrégés | nouvel expéditeur légitime, redirecteur ou abus | déterminer le volume, un exemple d’en-tête, l’attribution Reverse/fournisseur et le propriétaire interne du service |
| les `Authentication-Results` se contredisent | plusieurs sauts de vérification ou champ externe falsifié | utiliser uniquement les résultats au sein de la limite de confiance définie |
| `p=reject`, mais le message est distribué | dérogation locale chez le destinataire | vérifier `disposition`, le motif de dérogation et les autres résultats de filtrage |

## Histoire technique

SPF est issu de plusieurs propositions d’autorisation d’expéditeur SMTP et a été publié en 2006 comme RFC 4408 expérimentale. La RFC 7208 a fait passer SPF sur la voie de normalisation en 2014, supprimé le type DNS RR SPF séparé et précisé notamment les limites DNS et Void ([RFC 4408](https://datatracker.ietf.org/doc/html/rfc4408), [RFC 7208, annexe B](https://datatracker.ietf.org/doc/html/rfc7208#appendix-B)). Son architecture est restée délibérément orientée vers la connexion SMTP et le Reverse Path.

DKIM a réuni l’expérience de DomainKeys et d’Identified Internet Mail. La RFC 4871 a standardisé DKIM en 2007 ; la RFC 6376 l’a remplacée en 2011 et a précisé le modèle de signature, de clé et de vérification. La RFC 8301 a actualisé les algorithmes et longueurs de clé RSA en 2018, et la RFC 8463 a ajouté Ed25519-SHA256 ([RFC 4871](https://datatracker.ietf.org/doc/html/rfc4871), [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376), [RFC 8301](https://datatracker.ietf.org/doc/html/rfc8301), [RFC 8463](https://datatracker.ietf.org/doc/html/rfc8463)).

DMARC a d’abord été publié en 2015 avec la RFC 7489 comme document informatif. La RFC 9989 a remplacé en 2026 la RFC 7489 et l’extension PSD RFC 9091 comme spécification sur la voie de normalisation ; le reporting agrégé et le reporting d’échec ont simultanément été déplacés dans les RFC 9990 et RFC 9991. Ce changement a notamment introduit DNS Tree Walk, `np`, `psd` et `t`, et supprimé `pct` ([RFC 9989, annexe C](https://datatracker.ietf.org/doc/html/rfc9989#appendix-C), [RFC 9990](https://datatracker.ietf.org/doc/html/rfc9990), [RFC 9991](https://datatracker.ietf.org/doc/html/rfc9991)).

La transmission lisible par machine des résultats de vérification a évolué de la RFC 5451, via les RFC 7001 et 7601, vers la RFC 8601. ARC a été publié en 2019 avec la RFC 8617 comme tentative expérimentale de transmettre de manière signée les résultats d’authentification de flux indirects ([RFC 8601, section 6](https://datatracker.ietf.org/doc/html/rfc8601#section-6), [RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)). Cette histoire explique pourquoi les plateformes réelles peuvent présenter simultanément d’anciennes balises DMARC, des Organizational Domains basées sur PSL, différents algorithmes DKIM et des modèles de confiance Auth-Results différents.

## Sources

- [RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) – identités SPF, évaluation des enregistrements, résultats et limites DNS.
- [RFC 6376 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc6376) – modèle de signature, de clé et de vérification DKIM.
- [RFC 9989 – Domain-Based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc9989) – protocole central DMARC, alignement, politique, DNS Tree Walk et exploitation.
- [RFC 5321, sections 3.3 et 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 5322, section 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 8301 – DKIM Cryptographic Algorithm and Key Usage Update](https://datatracker.ietf.org/doc/html/rfc8301) – SHA-256 et longueurs de clé RSA.
- [RFC 8463 – Ed25519-SHA256 for DKIM](https://datatracker.ietf.org/doc/html/rfc8463) – algorithme de signature et de clé supplémentaire.
- [RFC 9990 – DMARC Aggregate Reporting](https://datatracker.ietf.org/doc/html/rfc9990) – modèle de données XML, transport, doublons et destinations de rapport externes.
- [RFC 9991 – DMARC Failure Reporting](https://datatracker.ietf.org/doc/html/rfc9991) – Failure Reports par message et protection des données.
- [RFC 8601 – Authentication-Results](https://datatracker.ietf.org/doc/html/rfc8601) – format d’en-tête, `authserv-id` et limite de confiance.
- [RFC 8617 – Authenticated Received Chain](https://datatracker.ietf.org/doc/html/rfc8617) – chaîne ARC expérimentale pour les flux indirects.
- [RFC 7960, sections 3 et 4](https://datatracker.ietf.org/doc/html/rfc7960)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – requêtes DNS sous Windows.
- [BIND 9 – manuel dig](https://bind9.readthedocs.io/en/latest/manpages.html) – requêtes DNS sous les systèmes Unix.
- [Microsoft Learn – Get-Content](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) – analyse de fichiers locaux et d’en-têtes sous Windows.
- [Microsoft Learn – Select-String](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) – analyse de motifs dans PowerShell.
- [manuel GNU grep](https://www.gnu.org/software/grep/manual/grep.html) – recherche de texte sous Linux et Unix.
- [libxml2 – xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) – vérification XML et XPath.
- [RFC 4408 – Sender Policy Framework, Experimental](https://datatracker.ietf.org/doc/html/rfc4408) – prédécesseur de la RFC 7208.
- [RFC 4871 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc4871) – prédécesseur de la RFC 6376.
