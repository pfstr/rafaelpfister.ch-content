---
title: "Licences : droits d’utilisation, métriques et points de comptage techniques"
blatt: "lizenzierung"
description: "Licences techniques pour les administrateurs d’infrastructure et de messagerie : droits, métriques d’utilisateurs, d’appareils, d’instances, de cœurs et de capacité, éditions, abonnements, activation, serveurs de licences et API cloud, sources d’identité, HA/DR, limites techniques d’application, données d’audit, Open Source et vérification."
fakten:
  - label: Déterminant
    wert: Contrat, Product Terms et commande ; l’affichage technique est un élément de preuve, pas le contrat juridique
    href: https://www.iso.org/standard/52293.html
  - label: Trois quantités
    wert: droits acquis · attribués techniquement · réellement utilisés
    href: https://www.iso.org/standard/52293.html
  - label: Métriques
    wert: utilisateur · appareil · instance · cœur/vCPU · capacité · transaction · fonctionnalité
    href: https://www.iso.org/standard/52293.html
  - label: Point de comptage
    wert: périmètre + ancre d’objet + filtre d’état + fenêtre temporelle + règle d’agrégation
    href: https://www.iso.org/standard/68531.html
  - label: ID logiciel
    wert: Les balises SWID standardisent l’identification du produit, mais pas automatiquement le droit d’utilisation
    href: https://www.iso.org/standard/65666.html
  - label: Licences basées sur l’identité
    wert: évaluer séparément les attributions directes et basées sur les groupes, le SKU et le plan de service
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA et DR
    wert: évaluer les nœuds passifs, le Cold Standby, les instances de test et de récupération uniquement selon les Terms concrets
    href: https://www.iso.org/standard/52293.html
  - label: Application
    wert: avertissement, limite de fonctionnalité, aucune nouvelle attribution, Grace Period ou interruption de service sont spécifiques au produit
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Fonctionnement hors ligne
    wert: documenter la durée des tokens/leasings, la Grace Period, l’horloge, le Trust Store et le chemin de récupération
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Open Source
    wert: utilisable gratuitement ne signifie pas sans obligations ; vérifier le texte de licence et le scénario de distribution
    href: https://opensource.org/osd
  - label: Lisible par machine
    wert: Les expressions SPDX modélisent les licences individuelles, alternatives et combinées
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Preuve administrateur
    wert: exporter de manière reproductible le droit, l’inventaire, l’attribution, l’utilisation, l’exception, le temps et la source
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: fb38ff7080a593642081c7fa017dd2d33735c4fd96cdf1ae256bebe4be37eb3d
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:39:31.070Z
translationReview: required
---

# Licences : droits d’utilisation, métriques et points de comptage techniques

La licence logicielle associe un contrat juridique d’utilisation à des objets mesurables techniquement. Pour les administrateurs, il est essentiel de ne pas confondre ces niveaux. Une clé de licence, un affichage cloud ou un compteur interne d’utilisateurs peut activer des fonctions et mesurer l’utilisation ; il ne définit toutefois pas à lui seul ce qu’une organisation est juridiquement autorisée à utiliser. ISO/IEC 19770-3 traite explicitement les données numériques de droits comme une représentation des droits d’utilisation et précise que les conditions de licence initiales prévalent à des fins juridiques ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La tâche opérationnelle consiste donc à **rapprocher de manière reproductible les droits acquis, les droits techniquement installés ou attribués et l’utilisation réelle**. Des écarts peuvent résulter d’une surutilisation, de coûts inutilisés, de filtres d’annuaire erronés, de comptes orphelins, de nœuds de cluster passifs, de tokens expirés ou simplement de définitions différentes du même mot « utilisateur ». Cet article décrit la perspective technique ; l’interprétation contractuelle et juridique relève des achats, de la gestion des licences et du conseil juridique.

L’explication commence par le droit d’utilisation contractuel et suit la manière dont le produit, la métrique et le point de comptage technique en font une consommation mesurée. Viennent ensuite l’attribution, l’application, l’audit, les cas particuliers et la récupération.

Une licence est d’abord un droit d’utilisation ; elle ne devient techniquement visible que par l’attribution, l’utilisation mesurée et, le cas échéant, l’application. L’article distingue ces quatre niveaux avant d’aborder les métriques produit et les audits.

## Droit, attribution, utilisation et application

Un bilan de licences rigoureux conserve quatre états indépendants :

1. **Droit (Entitlement) :** quels droits d’utilisation ont été acquis avec quel contrat, produit, métrique, périmètre, période et droit particulier ?
2. **Déploiement/attribution :** sur quels appareils, instances, utilisateurs, tenants ou fonctionnalités le logiciel est-il installé, activé ou attribué ?
3. **Utilisation :** quels objets ou fonctions pertinents pour la licence ont effectivement été utilisés pendant la période de mesure convenue ?
4. **Application (Enforcement) :** quelle limite le produit contrôle-t-il techniquement, et comment réagit-il en cas d’absence de connexion, d’expiration ou de dépassement ?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

Un droit existant ne prouve pas une attribution correcte ; une attribution ne prouve pas l’utilisation ; une faible utilisation n’annule pas automatiquement une licence utilisateur nommée. Inversement, un produit peut continuer à fonctionner techniquement alors qu’un droit d’abonnement ou de support a expiré. La conformité et la disponibilité technique sont donc deux objectifs de contrôle différents.

ISO/IEC 19770-1 spécifie les exigences d’un système de gestion des actifs informatiques. ISO/IEC 19770-2 standardise les Software Identification Tags (SWID), qui rendent les logiciels identifiables, mais qui, selon la norme, n’imposent pas de rapprochement des droits. ISO/IEC 19770-3 définit des termes et un format de transport pour les droits et les métriques associées. Ensemble, elles fournissent les niveaux de données **inventaire**, **identité logicielle** et **droit d’utilisation**, et non un calcul universel de licences ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Métriques de licence : que compte-t-on ?

Une métrique n’est interprétable qu’avec son contrat complet :

| Métrique | Ancre de comptage possible | Questions pour les administrateurs |
|---|---|---|
| Utilisateur nommé | ID immuable de personne/de tenant | les comptes désactivés, partagés, externes, de service ou de test comptent-ils ; peut-elle être réattribuée ? |
| Utilisateur/session simultané(e) | session active ou lease de checkout | quelle session commence/se termine, comment les délais d’expiration, appareils multiples et leases hors ligne sont-ils traités ? |
| Appareil | ID matériel, appareil enregistré, instance client | les VDI, appareils de remplacement, appareils partagés et agents nouvellement installés comptent-ils séparément ? |
| Serveur/instance/nœud | VM, hôte, appliance, nœud de cluster, instance de conteneur | les instances passives, temporaires, autoscalées, de test ou de récupération comptent-elles ? |
| Processeur/cœur/vCPU | socket/cœur physique, vCPU attribuée, nombre minimal | comment sont calculés l’Hyperthreading, l’affinité, les déplacements de cluster, les tailles cloud et les packs minimums ? |
| Capacité | boîtes aux lettres, stockage, domaines, messages, débit ou enregistrements | le pic, la moyenne, le maximum mensuel, le provisionné ou l’utilisé s’applique-t-il ; quelle fenêtre temporelle ? |
| Fonctionnalité/édition | plan de service activé, module, limite de base de données, API | l’installation, l’activation, la configuration ou l’utilisation réelle suffit-elle ? |
| Abonnement/consommation | attribution de SKU, crédits, requêtes, Go-mois | quand la consommation est-elle réservée, consommée, facturée ultérieurement ou restituée ? |

L’unité technique ne doit pas être devinée à partir du nom du produit. « Par cœur » peut désigner les cœurs physiques de l’hôte, les vCPU de la VM ou une métrique de cœurs normalisée. « Utilisateur » peut désigner une personne physique, un compte actif, une boîte aux lettres, une identité sous licence ou un expéditeur émetteur. « Instance » peut être comptée lorsqu’elle est en cours d’exécution, installée, enregistrée ou pour chaque nœud du cluster. ISO/IEC 19770-3 standardise une structure de droits, mais ne remplace pas la définition concrète dans les Product Terms et la commande ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La métrique seule n’indique pas encore où le comptage est effectué. Seul le point de comptage technique explique pourquoi le portail du fabricant, l’inventaire local et la valeur facturée peuvent différer.

## Le point de comptage technique

Chaque métrique est documentée comme une fonction de comptage reproductible :

**Compteur = périmètre × ancre d’objet × filtre d’état × fenêtre temporelle × règle d’agrégation × exceptions**

- **Périmètre :** organisation, tenant, domaine, cluster, abonnement, site ou contrat.
- **Ancre d’objet :** ID utilisateur immuable, ID appareil, ID VM, numéro de série d’hôte, ID SKU ou hash – et pas seulement un nom d’affichage.
- **Filtre d’état :** enabled, assigned, provisioned, active, seen, mounted, running ou consumed.
- **Fenêtre temporelle :** date de référence, mois civil, pic, moyenne, fenêtre glissante ou année contractuelle.
- **Agrégation :** distinct, somme, maximum, 95e percentile, pack minimum ou palier.
- **Exceptions :** utilisateurs externes, comptes système, instance DR passive, essai, NFR ou cas d’utilisation contractuellement gratuit.

Le point de comptage est souvent un **point de transfert**. Une importation LDAP peut compter tous les comptes correspondants, même s’ils n’utilisent jamais une fonction de messagerie. Une passerelle peut apprendre des expéditeurs à partir du trafic et ainsi conserver des identités supprimées ou techniques. Un SKU cloud peut être attribué directement ou indirectement via un groupe. L’API `licenseDetails` de Microsoft Graph fournit les licences directes et héritées d’une appartenance à un groupe, ainsi que les plans de service individuels et les états de provisionnement ; cela montre pourquoi un booléen « a une licence » ne suffit pas à l’analyse ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Architecture d’un système de licences

Les produits commerciaux utilisent des implémentations différentes, mais peuvent être techniquement décomposés en surfaces de contrôle. Ce modèle est une synthèse opérationnelle des niveaux de données ISO/IEC 19770 et de mécanismes documentés par les fabricants, et non une architecture normative universelle :

| Surface de contrôle | Fonction | Domaine de défaillance |
|---|---|---|
| Magasin de droits | contrat/SKU, quantité, période, fonctionnalités, droits particuliers | commande erronée, expirée, mauvais tenant/Smart Account |
| Source d’inventaire/d’identité | utilisateurs, appareils, instances, cœurs, clusters, ID logiciel | doublon, objet obsolète, filtre ou périmètre erroné |
| Attribution | attribue un droit à un objet ou à un plan de service | attribution directe ou basée sur un groupe, erreur de provisionnement |
| Compteur | collecte l’utilisation, les sessions, la capacité ou les heartbeats | temps, tampon hors ligne, échantillonnage, télémétrie manquante |
| Évaluateur | applique la métrique, le pool, les exceptions et la période | logique contractuelle ne correspondant pas à la politique technique |
| Service de licences | serveur local, API cloud, token, lease, certificat ou clé | DNS/TLS/proxy/horloge/trust, panne ou limite de débit |
| Application | active l’édition/la fonctionnalité ou limite le comportement | blocage dur, Grace Period, avertissement ou fail-open/fail-closed |
| Export des preuves | données d’audit, d’utilisation et d’attribution | rotation, protection des données, historique manquant ou export non reproductible |

Cisco Smart Licensing décrit une gestion centralisée des comptes et des licences ; Microsoft Graph expose les SKU acquis, les attributions et les plans de service via des API. Ces systèmes font des licences une dépendance distribuée entre identité, service cloud, réseau, TLS et temps. Un produit peut continuer à traiter des données tout en ne pouvant plus obtenir de nouvelle licence ; un autre peut bloquer des fonctionnalités après un délai hors ligne. Le comportement concret doit être repris de la documentation du fabricant et du contrat ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licences basées sur l’identité

Pour les utilisateurs nommés, l’annuaire fait partie du système de licences. Un filtre correct ne répond pas seulement à « personnes dans l’OU X », mais aussi aux questions suivantes :

- quel attribut constitue l’ancre d’objet immuable ;
- si les comptes désactivés, verrouillés, supprimés ou pas encore provisionnés comptent ;
- comment sont traitées les boîtes aux lettres partagées/de ressources, les comptes de service, les invités, les partenaires externes et les comptes de test ;
- si les attributions directes et basées sur des groupes sont fusionnées ;
- quand une licence retirée redevient disponible ;
- si l’utilisation historique ou seulement l’état à la date de référence est déterminant ;
- quels attributs le produit met en cache localement et quand il les supprime.

LDAP fournit des entrées et des attributs, mais pas de définition universelle de « personne sous licence ». [LDAP](/kb/ldap) peut techniquement mettre en œuvre le périmètre et le filtre ; la métrique découle du contrat. Il en va de même dans les annuaires cloud : SKU, plan de service, `assignedLicenses`, état de provisionnement et état réel de la charge de travail sont des données différentes. Microsoft Graph documente `subscribedSku` comme abonnement commercial acquis et `licenseDetails` pour chaque utilisateur ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

Le nettoyage des comptes orphelins ne doit pas être directement motivé par la pression sur les licences. Il convient d’abord de vérifier les propriétaires, la conservation, le routage de messagerie, le Legal Hold, les dépendances de service et la récupération ; l’identité est ensuite désactivée ou supprimée de manière contrôlée. Un rapport de licences est une entrée dans le cycle de vie, pas une instruction de suppression.

## Modèles d’instance, de cœur et de capacité

La virtualisation et les clusters rendent le nom du serveur physique insuffisant. Un inventaire nécessite l’hôte, la VM/le conteneur, les vCPU attribuées, les processeurs/cœurs physiques, le cluster d’hyperviseurs, les règles de mobilité, l’édition et le rôle. Si une VM peut se déplacer entre hôtes, l’ensemble du périmètre d’hôtes possible peut être pertinent selon le contrat ; une affinité CPU stricte peut être prouvée techniquement, sans être automatiquement reconnue contractuellement.

Les éditions associent le droit d’utilisation à des limites techniques. Microsoft documente par exemple pour Exchange des éditions qui diffèrent notamment par le nombre de bases de données montées simultanément ; les copies de bases de données passives peuvent également compter comme bases de données montées. La clé de produit définit l’édition du serveur. C’est un exemple de convergence entre application et architecture de capacité, et non un calcul général de licences Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Les métriques de capacité nécessitent un historique temporel. Une valeur instantanée ne montre ni le pic mensuel ni un dépassement de courte durée. Pour la messagerie, les boîtes aux lettres actives, les expéditeurs internes, les domaines, le nombre quotidien de messages, le débit, le stockage et les utilisateurs chiffrés sont notamment des compteurs techniques courants – seul le droit du produit détermine s’ils sont pertinents pour la licence. Les tableaux de bord conservent donc la valeur brute, le temps, le périmètre, la source et la règle de calcul plutôt qu’un simple feu tricolore.

Dès lors que les instances ou les identités sont comptées, la haute disponibilité, les tests et les migrations concernent également le volume de licences. Les systèmes passifs ne sont pas automatiquement gratuits ; le contrat applicable est déterminant.

## HA, Disaster Recovery, test et migration

Les nœuds passifs, le Cold Standby, les instances de récupération, les laboratoires, les tests, les formations et les états parallèles temporaires pendant une [migration](/kb/migration) sont des cas particuliers typiques. Techniquement, « passif » peut néanmoins signifier que le logiciel est installé, que la réplication est traitée, que des bases de données sont montées ou qu’une licence est checkoutée par le serveur. Les termes contractuels doivent être mappés sur des états observables.

Pour chaque cas particulier, les éléments suivants sont documentés :

- le nombre autorisé et la définition des instances passives/froides ;
- si le test, la qualification de correctifs, la restauration de sauvegarde ou le test DR sont inclus ;
- combien de temps l’exploitation simultanée de l’ancien et du nouveau système est autorisée lors d’une migration ;
- si une licence est mobile et quelles conditions de réattribution/d’attente s’appliquent ;
- si les droits cloud et On-premises sont liés ;
- quelle preuve distingue un véritable cas DR d’une charge durable de production ;
- comment le service de licences fonctionne dans un réseau de récupération isolé.

Le [concept de sauvegarde et de DR](/kb/backup-dr) contient donc également les fichiers de licence, les données d’activation, les serveurs de licences, les tokens hors ligne, les certificats, l’heure, le DNS/proxy et les contacts du fabricant. Une restauration techniquement parfaite qui ne peut pas activer son édition ou les fonctionnalités requises n’atteint pas l’objectif de récupération.

## Expiration, Grace Period et panne du service de licences

Dans les modèles d’abonnement, de lease ou cloud, les temporisateurs suivants sont au minimum pertinents : fin du droit, expiration du token, heartbeat, durée de borrow/checkout, délai hors ligne, Grace Period et validité du certificat. La perspective administrateur conserve **l’heure absolue, le fuseau horaire, la source de synchronisation et le dernier renouvellement réussi**. Une horloge erronée peut déclencher une apparente expiration de licence ou empêcher la validation d’un token signé.

Le comportement à l’expiration est spécifique au produit :

- simple avertissement ou événement de conformité ;
- aucune nouvelle attribution, l’utilisation existante reste possible ;
- fonctionnalité Premium désactivée ou retour à une édition inférieure ;
- nombre limité de nouvelles sessions/utilisateurs ;
- fonctionnement en lecture seule ;
- interruption complète du service ;
- Grace Period locale lorsque le service cloud n’est pas joignable.

Cette réaction est déterminée dans un environnement de test ou à partir d’une source explicite du fabricant, et non testée dans le flux de production. La supervision alerte sur le premier temporisateur opérationnel, et pas seulement sur la fin du contrat. Cisco documente ses propres mécanismes en ligne/hors ligne et de compte pour Smart Licensing ; d’autres fabricants utilisent des serveurs de licences locaux, des fichiers signés, des dongles USB, des clés de produit ou des droits SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Open Source : droit d’utilisation sans compteur technique

Open Source ne signifie pas simplement code source visible. L’Open Source Definition exige notamment la libre redistribution, l’accès au code source, les œuvres dérivées et des droits neutres sur le plan technologique. Les licences individuelles imposent toutefois différentes conditions concernant les modifications, la redistribution, les notices, la fourniture du code source ou les licences de brevets ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

L’Apache License 2.0 accorde des licences de copyright et de brevets sous conditions et exige notamment, lors de la redistribution, une copie de la licence, des indications de modification et le maintien de certaines notices. Un prix de téléchargement de zéro ne supprime pas ces obligations ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). Pour les ensembles de composants, plusieurs licences peuvent s’appliquer simultanément ou alternativement.

Les SPDX License Expressions modélisent ces cas de manière lisible par machine : `AND` pour les obligations cumulatives, `OR` pour un choix de licence et `WITH` pour une exception. Une expression SPDX identifie la situation de licence déclarée, mais n’effectue pas de contrôle de compatibilité juridique ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Pour les administrateurs, les fichiers de licence et de notices, la SBOM/la liste des composants, le canal de distribution et les propres modifications font partie des preuves de publication et d’archivage.

## Outils d’administration pour des compteurs reproductibles

Les exemples montrent des requêtes techniques d’inventaire et de preuve. La pertinence d’un champ pour les licences doit être déduite des Terms. Les exports peuvent contenir des données personnelles et nécessitent une protection d’accès, une conservation et une limitation des finalités.

### Collecter l’inventaire CPU, cœurs et virtualisation

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) lit les données d’inventaire CIM/WMI ; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) collecte les données CPU, cœurs, threads, sockets et NUMA. Dans une VM, elles montrent principalement la vue de l’invité. Le périmètre de l’hôte, les déplacements de cluster, le type d’instance cloud et les facteurs de cœurs contractuels sont documentés séparément.

### Compter les objets d’annuaire avec un périmètre explicite

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) et [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) fournissent des ancres d’objet et des attributs. La Search Base, le filtre, la pagination, les attributs à valeurs multiples et les objets désactivés sont documentés. L’export ne compte pas des « personnes soumises à licence » tant que la règle contractuelle n’est pas mappée exactement sur ces champs.

### Lire les SKU cloud et les attributions aux utilisateurs

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus
  </code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) et [`curl`](https://curl.se/docs/manpage.html) lisent les données Microsoft Graph. Les tokens ne sont pas stockés dans des scripts, tickets ou l’historique du shell. `ConsumedUnits`, l’attribution à l’utilisateur, le provisionnement du plan de service et la charge de travail effectivement utilisée sont évalués séparément.

### Vérifier le serveur de licences ou l’endpoint cloud depuis le réseau produit

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient l’établissement DNS/TCP depuis l’origine choisie. Un succès ne prouve ni le proxy, [TLS](/kb/tls), le token, l’attribution de compte ni la transaction de licence. Les journaux produit et l’état du fabricant fournissent la transition d’état suivante.

### Protéger l’export d’audit contre les modifications

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) détectent les modifications ultérieures d’octets. La date de création, le périmètre de requête, la version de l’outil/de l’API, la requête, le fuseau horaire, l’exportateur et le stockage sécurisé sont également documentés. Un hash ne confirme pas l’exhaustivité fonctionnelle.

### Vérifier les événements de licence et d’activation durant la période

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) et [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrent les événements locaux. Les journaux du fabricant peuvent utiliser des codes structurés plutôt que des messages texte en anglais ; il est préférable d’utiliser le fournisseur, l’ID d’événement ou des champs définis. L’heure, la rotation et la copie centralisée des journaux font partie du constat.

La preuve d’audit résulte du droit, du point de comptage et de l’état opérationnel. Elle doit montrer de manière reproductible ce qui a été acheté, attribué, installé et effectivement utilisé.

## Preuve d’audit et d’exploitation

Une preuve de licence reproductible contient :

- référence du contrat/de la commande, produit/SKU, métrique, quantité, périmètre, durée et droits particuliers ;
- inventaire logiciel et matériel avec IDs immuables, édition, version et relation de cluster ;
- attributions directes, basées sur des groupes et automatiques avec source et état de provisionnement ;
- valeurs de mesure brutes et règle de calcul pour chaque fenêtre temporelle ;
- exceptions avec justification, propriétaire, date d’expiration et contrôle technique ;
- HA/DR/test/migration comme population distincte ;
- affichage produit, export API/CLI et contre-calcul indépendant ;
- heure, fuseau horaire, version de requête/d’outil, hash et stockage protégé.

Les écarts sont classifiés : **erreur de données** (doublons, anciens comptes), **erreur de modèle** (mauvaise métrique), **erreur de processus** (licence non supprimée après un départ), **erreur technique** (synchronisation/token/serveur) ou **droit manquant**. Cette distinction seule permet de savoir si la réponse est le nettoyage, la configuration, la clarification contractuelle, l’acquisition ou la réponse à incident.

## Évolution technique des licences

Les clés de produit locales et les dongles ont d’abord associé étroitement le droit d’utilisation à une machine. Les serveurs de licences réseau ont introduit des pools partagés et des leases simultanés ; la virtualisation a exigé de nouvelles règles d’hôtes, de cœurs et de mobilité. Les abonnements et le SaaS ont déplacé les droits vers des comptes cloud, des groupes d’identité et des plans de service. Cisco Smart Licensing et Microsoft Graph illustrent des modèles centralisés de comptes/API, tandis qu’ISO/IEC 19770 standardise les données SWID et de droits ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallèlement, l’Open Source a rendu l’activation technique superflue pour de nombreux composants, mais pas les conditions de licence. SPDX a créé des identifiants et expressions abrégés lisibles par machine pour les chaînes logicielles. La tâche de l’administrateur est ainsi passée de « saisir une clé » à un problème de données portant sur le contrat, l’identité, l’actif, la durée, la télémétrie et la composition logicielle. Le contre-modèle est le même que pour la supervision : ancres d’objet claires, fenêtres temporelles explicites, données brutes et calculs reproductibles.

## Sources

- [ISO/IEC 19770-3:2016 – Schéma de droits](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – Systèmes de gestion des actifs informatiques](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Balise d’identification logicielle](https://www.iso.org/standard/65666.html)
- [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)
- [Cisco – Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)
- [Microsoft Graph PowerShell – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)
- [Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions)
- [Microsoft – Enter Exchange Server Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)
- [Open Source Initiative – Open Source Definition](https://opensource.org/osd)
- [Apache Software Foundation – Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [SPDX Specification 3.0.1 – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)
- [Microsoft Learn – Get-CimInstance](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance)
- [Linux man-pages – lscpu](https://man7.org/linux/man-pages/man1/lscpu.1.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Graph PowerShell – Get-MgUserLicenseDetail](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail)
- [curl – page de manuel de ligne de commande](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – Utilitaires SHA-2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [GNU Grep – Manuel](https://www.gnu.org/software/grep/manual/grep.html)
