---
title: "Licences : droits d’utilisation, métriques et points de comptage techniques"
blatt: "lizenzierung"
description: "Licences techniques pour les administrateurs d’infrastructure et de messagerie : droits, métriques d’utilisateurs, d’appareils, d’instances, de cœurs et de capacité, éditions, abonnements, activation, serveurs de licences et API cloud, sources d’identité, HA/DR, limites techniques d’application, données d’audit, Open Source et vérification."
fakten:
  - label: Déterminant
    wert: Contrat, Product Terms et commande ; l’affichage technique est une preuve, non le contrat juridique
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
    wert: Les balises SWID standardisent l’identification des produits, pas automatiquement le droit d’utilisation
    href: https://www.iso.org/standard/65666.html
  - label: Licences basées sur l’identité
    wert: évaluer séparément l’attribution directe et par groupe, la SKU et le plan de service
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA et DR
    wert: évaluer les nœuds passifs, Cold Standby, instances de test et de reprise uniquement selon les Terms concrets
    href: https://www.iso.org/standard/52293.html
  - label: Application
    wert: avertissement, limite de fonctionnalité, aucune nouvelle attribution, Grace Period ou interruption de service sont spécifiques au produit
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Fonctionnement hors ligne
    wert: documenter la durée des tokens/leasings, la Grace Period, l’horloge, le Trust Store et le chemin de reprise
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Open Source
    wert: utilisable gratuitement ne signifie pas sans obligations ; vérifier le texte de licence et le scénario de distribution
    href: https://opensource.org/osd
  - label: Lisible par machine
    wert: les expressions SPDX modélisent les licences individuelles, alternatives et combinées
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Preuve administrateur
    wert: exporter de façon reproductible entitlement · inventaire · attribution · utilisation · exception · heure · source
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: 4abc9d3d627b9f38adb1f2fcb541662b430f07694209251464f0a4b0f9c4351f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:29:25.356Z
translationReview: required
---

# Licences : droits d’utilisation, métriques et points de comptage techniques

La licence logicielle associe un contrat juridique d’utilisation à des objets techniquement mesurables. Pour les administrateurs, il est essentiel de ne pas confondre ces niveaux. Une clé de licence, un affichage cloud ou un compteur d’utilisateurs interne peut activer des fonctionnalités et mesurer l’utilisation ; il ne définit toutefois pas à lui seul ce qu’une organisation est juridiquement autorisée à utiliser. La norme ISO/IEC 19770-3 traite explicitement les données d’entitlement numériques comme une représentation des droits d’utilisation et précise que les conditions de licence originales prévalent à des fins juridiques ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La tâche opérationnelle consiste donc à **comparer de manière reproductible les droits acquis, les droits techniquement installés ou attribués et l’utilisation réelle**. Les écarts peuvent signaler une surutilisation, des coûts inutilisés, des filtres d’annuaire erronés, des comptes orphelins, des nœuds de cluster passifs, des tokens expirés ou simplement des définitions différentes du même terme « utilisateur ». L’article décrit la perspective technique ; l’interprétation contractuelle et juridique relève des achats, de la gestion des licences et du conseil juridique.

L’explication commence par le droit d’utilisation contractuel et suit la manière dont le produit, la métrique et le point de comptage technique en font une consommation mesurée. Elle aborde ensuite l’attribution, l’application, l’audit, les cas particuliers et la reprise.

Une licence est avant tout un droit d’utilisation ; elle ne devient techniquement visible que par l’attribution, l’utilisation mesurée et, le cas échéant, l’application. L’article distingue ces quatre niveaux avant d’aborder les métriques produit et les audits.

## Entitlement, attribution, utilisation et application

Un bilan de licences propre distingue quatre états indépendants :

1. **Entitlement :** Quels droits d’utilisation ont été acquis, avec quel contrat, produit, métrique, périmètre, période et droit particulier ?
2. **Déploiement/attribution :** Sur quels appareils, instances, utilisateurs, tenants ou fonctionnalités le logiciel est-il installé, activé ou attribué ?
3. **Utilisation :** Quels objets ou fonctionnalités pertinents pour la licence ont effectivement été utilisés durant la période de mesure convenue ?
4. **Application :** Quelle limite le produit contrôle-t-il techniquement et comment réagit-il en cas d’absence de connexion, d’expiration ou de dépassement ?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

Un entitlement existant ne prouve pas qu’une attribution est correcte ; une attribution ne prouve pas une utilisation ; une faible utilisation n’annule pas automatiquement une licence utilisateur nommée. Inversement, un produit peut continuer techniquement à fonctionner bien qu’un droit d’abonnement ou de support ait expiré. La conformité et la disponibilité technique sont donc deux objectifs de contrôle distincts.

La norme ISO/IEC 19770-1 spécifie les exigences d’un système de gestion des actifs IT. ISO/IEC 19770-2 standardise les Software Identification Tags (SWID), qui rendent les logiciels identifiables mais n’imposent pas, selon la norme, de réconciliation des entitlements. ISO/IEC 19770-3 définit des termes et un format de transport pour les entitlements et les métriques associées. Ensemble, elles fournissent les niveaux de données **inventaire**, **identité logicielle** et **droit d’utilisation**, et non un calcul universel des licences ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Métriques de licence : que compte-t-on ?

Une métrique ne peut être interprétée qu’avec son contrat complet :

| Métrique | Ancre de comptage possible | Questions pour les administrateurs |
|---|---|---|
| Utilisateur nommé | ID immuable de personne/tenant | les comptes désactivés, partagés, externes, de service ou de test comptent-ils ; peut-elle être réattribuée ? |
| Utilisateur/session simultané(e) | session active ou lease de checkout | quelle session commence ou se termine, comment les timeouts, appareils multiples et leases hors ligne sont-ils traités ? |
| Appareil | ID matériel, appareil enregistré, instance cliente | les VDI, appareils de remplacement, appareils partagés et agents nouvellement installés comptent-ils séparément ? |
| Serveur/instance/nœud | VM, hôte, appliance, nœud de cluster, instance de conteneur | les instances passives, temporaires, autoscalées, de test ou de reprise comptent-elles ? |
| Processeur/cœur/vCPU | socket/cœur physique, vCPU attribuée, nombre minimal | comment sont calculés l’hyperthreading, l’affinité, le déplacement de cluster, les tailles cloud et les packs minimaux ? |
| Capacité | boîtes aux lettres, stockage, domaines, messages, débit ou jeux de données | peak, moyenne, maximum mensuel, provisioned ou used s’appliquent-ils ; quelle fenêtre temporelle ? |
| Fonctionnalité/édition | plan de service, module, limite de base de données ou API activé(e) | l’installation, l’activation, la configuration ou l’utilisation effective suffit-elle ? |
| Abonnement/consommation | attribution de SKU, crédits, requêtes, Go-mois | à quel moment est-ce réservé, consommé, facturé ultérieurement ou restitué ? |

L’unité technique ne doit pas être déduite du nom du produit. « Par cœur » peut désigner les cœurs physiques de l’hôte, les vCPU de la VM ou une métrique de cœurs normalisée. « Utilisateur » peut désigner une personne physique, un compte actif, une boîte aux lettres, une identité licenciée ou un expéditeur. « Instance » peut être comptée en cours d’exécution, installée, enregistrée ou par nœud de cluster. ISO/IEC 19770-3 standardise une structure d’entitlement, mais ne remplace pas la définition concrète dans les Product Terms et la commande ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La métrique seule n’indique pas encore où le comptage a lieu. Seul le point de comptage technique explique pourquoi le portail de l’éditeur, l’inventaire local et la valeur facturée peuvent diverger.

## Le point de comptage technique

Chaque métrique est documentée sous forme de fonction de comptage reproductible :

**Compteur = Périmètre × Ancre d’objet × Filtre d’état × Fenêtre temporelle × Règle d’agrégation × Exceptions**

- **Périmètre :** organisation, tenant, domaine, cluster, abonnement, site ou contrat.
- **Ancre d’objet :** ID utilisateur immuable, ID appareil, ID VM, numéro de série d’hôte, ID SKU ou hash – pas seulement le nom d’affichage.
- **Filtre d’état :** enabled, assigned, provisioned, active, seen, mounted, running ou consumed.
- **Fenêtre temporelle :** date de référence, mois civil, peak, moyenne, fenêtre glissante ou année contractuelle.
- **Agrégation :** distinct, somme, maximum, 95e percentile, pack minimal ou palier.
- **Exceptions :** utilisateurs externes, comptes système, instance DR passive, Trial, NFR ou cas d’utilisation contractuellement gratuit.

Le point de comptage est souvent un **point de transfert**. Une importation LDAP peut compter tous les comptes correspondants, même s’ils n’utilisent jamais de fonction de messagerie. Une passerelle peut apprendre des expéditeurs à partir du trafic et ainsi conserver des identités supprimées ou techniques. Une SKU cloud peut être attribuée directement ou indirectement par un groupe. L’API `licenseDetails` de Microsoft Graph fournit les licences directes et héritées de l’appartenance à un groupe, ainsi que les plans de service individuels et leur état de provisionnement ; cela montre pourquoi un booléen « possède une licence » ne suffit pas à l’analyse ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Architecture d’un système de licences

Les produits commerciaux utilisent différentes implémentations, mais peuvent techniquement être décomposés en surfaces de contrôle. Ce modèle est une synthèse opérationnelle des niveaux de données ISO/IEC 19770 et des mécanismes documentés par les éditeurs, et non une architecture normative universelle :

| Surface de contrôle | Fonction | Domaine de défaillance |
|---|---|---|
| Entitlement Store | contrat/SKU, quantité, période, fonctionnalités, droits particuliers | commande erronée, expirée, mauvais tenant/Smart Account |
| Source d’inventaire/d’identité | utilisateurs, appareils, instances, cœurs, clusters, ID logiciel | doublon, objet obsolète, filtre ou périmètre erroné |
| Attribution | associe un droit à un objet ou à un plan de service | attribution directe ou par groupe, erreur de provisionnement |
| Compteur | collecte l’utilisation, les sessions, la capacité ou les heartbeats | temps, tampon hors ligne, échantillonnage, télémétrie absente |
| Évaluateur | applique métrique, pool, exceptions et période | la logique contractuelle ne correspond pas à la politique technique |
| Service de licences | serveur local, API cloud, token, lease, certificat ou clé | DNS/TLS/proxy/horloge/trust, panne ou Rate Limit |
| Application | active l’édition/la fonctionnalité ou limite le comportement | blocage dur, Grace Period, avertissement ou Fail-open/Fail-closed |
| Export de preuves | données d’audit, d’utilisation et d’attribution | rotation, protection des données, historique absent ou export non reproductible |

Cisco Smart Licensing décrit une gestion centralisée des comptes et licences ; Microsoft Graph expose les SKU acquises, les attributions et les plans de service via des API. Ces systèmes font de la licence une dépendance distribuée entre identité, service cloud, réseau, TLS et temps. Un produit peut continuer à traiter des données sans pouvoir obtenir de nouvelle licence ; un autre peut bloquer des fonctionnalités après une période hors ligne. Le comportement concret doit être repris de la documentation de l’éditeur et du contrat ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licences basées sur l’identité

Pour les utilisateurs nommés, l’annuaire fait partie du système de licences. Un filtre correct ne répond pas seulement à « personnes dans l’OU X », mais aussi aux questions suivantes :

- quel attribut constitue l’ancre d’objet immuable ;
- si les comptes désactivés, verrouillés, supprimés ou pas encore provisionnés comptent ;
- comment sont traités les boîtes aux lettres partagées/de ressources, comptes de service, invités, partenaires externes et comptes de test ;
- si les attributions directes et par groupe sont consolidées ;
- à quel moment une licence supprimée redevient disponible ;
- si l’utilisation historique ou seulement l’état à la date de référence est déterminant ;
- quels attributs le produit met en cache localement et à quel moment il les supprime.

LDAP fournit des entrées et attributs, mais pas de définition universelle de « personne licenciée ». [LDAP](/kb/ldap) peut techniquement appliquer le périmètre et le filtre ; la métrique provient du contrat. Il en va de même dans les annuaires cloud : SKU, plan de service, `assignedLicenses`, état de provisionnement et état réel de la charge de travail sont des données distinctes. Microsoft Graph documente `subscribedSku` comme abonnement commercial acquis et `licenseDetails` par utilisateur ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

Le nettoyage des comptes orphelins ne doit pas être directement guidé par la pression sur les licences. Il faut d’abord vérifier les propriétaires, la rétention, le routage de messagerie, le Legal Hold, les dépendances de service et la reprise ; l’identité est ensuite désactivée ou supprimée de manière contrôlée. Un rapport de licences est une entrée dans le cycle de vie, pas un ordre de suppression.

## Modèles par instance, cœur et capacité

La virtualisation et les clusters rendent insuffisant le nom du serveur physique. Un inventaire doit inclure l’hôte, la VM/le conteneur, les vCPU attribuées, le CPU/cœur physique, le cluster d’hyperviseurs, les règles de mobilité, l’édition et le rôle. Lorsqu’une VM peut se déplacer entre des hôtes, l’ensemble du périmètre potentiel des hôtes peut être pertinent selon le contrat ; une affinité CPU stricte peut être techniquement démontrable, mais n’est pas automatiquement reconnue contractuellement.

Les éditions associent le droit d’utilisation à des limites techniques. Microsoft documente par exemple pour Exchange Server des éditions qui diffèrent notamment par le nombre de bases de données montées simultanément ; les copies passives de bases de données peuvent également compter comme bases montées. La Product Key définit l’édition du serveur. C’est un exemple de convergence entre application et architecture de capacité, et non un calcul général de licences Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Les métriques de capacité nécessitent un historique. Une valeur instantanée ne montre ni le peak mensuel ni un dépassement temporaire. Pour la messagerie, les compteurs techniques courants comprennent notamment les boîtes aux lettres actives, les expéditeurs internes, les domaines, le nombre quotidien de messages, le débit, le stockage et les utilisateurs chiffrés ; seul le droit produit détermine s’ils sont pertinents pour la licence. Les tableaux de bord enregistrent donc la valeur brute, l’heure, le périmètre, la source et la règle de calcul plutôt qu’un simple feu tricolore.

Dès lors que les instances ou identités sont comptées, la haute disponibilité, les tests et les migrations affectent également le volume de licences. Les systèmes passifs ne sont pas automatiquement gratuits ; le contrat applicable fait foi.

## HA, Disaster Recovery, test et migration

Les nœuds passifs, Cold Standby, instances de reprise, laboratoires, tests, formations et états parallèles temporaires pendant une [migration](/kb/migration) constituent des cas particuliers typiques. Techniquement « passif » peut néanmoins signifier que le logiciel est installé, que la réplication est traitée, que des bases de données sont montées ou qu’une licence est checkout par le serveur. Les termes contractuels doivent être mis en correspondance avec des états observables.

Pour chaque cas particulier, il convient de documenter :

- le nombre autorisé et la définition des instances passives/froides ;
- si le test, la qualification de correctifs, la restauration de sauvegarde ou le test DR sont inclus ;
- combien de temps l’exploitation simultanée de l’ancien et du nouveau système est autorisée lors d’une migration ;
- si une licence est mobile et quelles conditions de réattribution/d’attente s’appliquent ;
- si les droits cloud et on-premises sont liés ;
- quelle preuve distingue un véritable cas DR d’une charge productive permanente ;
- comment le service de licences fonctionne dans un réseau de reprise isolé.

Le [concept de sauvegarde et de DR](/kb/backup-dr) contient donc aussi les fichiers de licence, données d’activation, serveurs de licences, tokens hors ligne, certificats, heure, DNS/proxy et contacts de l’éditeur. Une restauration techniquement parfaite qui ne peut activer son édition ou les fonctionnalités nécessaires n’atteint pas l’objectif de reprise.

## Expiration, Grace Period et panne du service de licences

Dans les modèles d’abonnement, de lease ou cloud, au moins les minuteries suivantes sont pertinentes : fin de l’entitlement, expiration du token, heartbeat, durée de Borrow/Checkout, délai hors ligne, Grace Period et validité du certificat. La vue administrateur consigne **l’heure absolue, le fuseau horaire, la source de synchronisation et le dernier renouvellement réussi**. Une horloge incorrecte peut déclencher une expiration de licence apparente ou empêcher la validation d’un token signé.

Le comportement à l’expiration est spécifique au produit :

- avertissement ou événement de conformité uniquement ;
- aucune nouvelle attribution, l’utilisation existante demeure ;
- fonctionnalité Premium désactivée ou retour à une édition inférieure ;
- nombre limité de nouvelles sessions/utilisateurs ;
- fonctionnement en lecture seule ;
- interruption complète du service ;
- Grace Period locale lorsque le service cloud est inaccessible.

Cette réaction est déterminée dans un environnement de test ou à partir d’une source explicite de l’éditeur, et non testée dans le déroulement de la production. La supervision alerte sur la première échéance opérationnelle, pas seulement sur la fin du contrat. Cisco documente pour Smart Licensing ses propres mécanismes en ligne/hors ligne et de compte ; d’autres éditeurs utilisent des serveurs de licences locaux, des fichiers signés, des dongles USB, des Product Keys ou des entitlements SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Open Source : droit d’utilisation sans compteur technique

Open Source ne signifie pas seulement code source visible. L’Open Source Definition exige notamment la libre redistribution, l’accès au code source, les œuvres dérivées et des droits neutres sur le plan technologique. Les licences individuelles imposent des conditions différentes en matière de modification, redistribution, notices, fourniture du code source ou licences de brevet ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

L’Apache License 2.0 accorde des licences de copyright et de brevet sous conditions et exige notamment, en cas de redistribution, une copie de la licence, des mentions de modification et le maintien de certaines notices. Un prix de téléchargement nul ne supprime pas ces obligations ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). Dans les ensembles de composants, plusieurs licences peuvent s’appliquer simultanément ou alternativement.

Les SPDX License Expressions modélisent ces cas de manière lisible par machine : `AND` pour les obligations cumulatives, `OR` pour un choix de licence et `WITH` pour une exception. Une expression SPDX identifie la situation de licence déclarée, mais n’effectue aucune vérification de compatibilité juridique ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Pour les administrateurs, les fichiers de licence et de notices, la SBOM/liste de composants, le canal de distribution et leurs propres modifications font partie de la preuve de release et d’archivage.

## Outils d’administration pour des compteurs reproductibles

Les exemples montrent des requêtes techniques d’inventaire et de preuves. La pertinence d’un champ pour une licence doit être déduite des Terms. Les exports peuvent contenir des données personnelles et nécessitent contrôle d’accès, rétention et limitation de la finalité.

### Recenser l’inventaire CPU, cœurs et virtualisation

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-CimInstance Win32_ComputerSystem |
  Select-Object Name,Manufacturer,Model,NumberOfProcessors,NumberOfLogicalProcessors,HypervisorPresent
Get-CimInstance Win32_Processor |
  Select-Object DeviceID,Name,SocketDesignation,NumberOfCores,NumberOfLogicalProcessors</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">lscpu --extended=CPU,NODE,SOCKET,CORE,ONLINE
lscpu</code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) lit les données d’inventaire CIM/WMI ; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) collecte les données CPU, cœurs, threads, sockets et NUMA. Dans une VM, elles montrent principalement la vue de l’invité. Le périmètre hôte, le déplacement de cluster, le type d’instance cloud et les facteurs de cœurs contractuels sont documentés séparément.

### Compter les objets d’annuaire avec un périmètre explicite

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -SearchBase 'OU=Licensed,DC=example,DC=ch' -Filter * \
  -Properties ObjectGUID,Enabled,mail,employeeType |
  Select-Object ObjectGUID,SamAccountName,Enabled,mail,employeeType |
  Export-Csv .\\licensed-users.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'ou=Licensed,dc=example,dc=ch' \
  '(&(objectClass=person)(mail=*))' entryUUID uid mail employeeType</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) et [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) fournissent des ancres d’objet et des attributs. La Search Base, le filtre, la pagination, les attributs à valeurs multiples et les objets désactivés sont documentés. L’export ne compte pas les « personnes soumises à licence » tant que la règle contractuelle n’est pas exactement mappée sur ces champs.

### Lire les SKU cloud et les attributions utilisateur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-MgSubscribedSku -All |
  Select-Object SkuId,SkuPartNumber,ConsumedUnits,PrepaidUnits,CapabilityStatus
Get-MgUserLicenseDetail -UserId 'admin@example.ch' |
  Select-Object SkuId,SkuPartNumber,ServicePlans</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) et [`curl`](https://curl.se/docs/manpage.html) lisent les données Microsoft Graph. Les tokens ne sont pas enregistrés dans les scripts, tickets ou historiques shell. `ConsumedUnits`, l’attribution utilisateur, le provisionnement du plan de service et la charge de travail réellement utilisée sont évalués séparément.

### Vérifier le serveur de licences ou le point de terminaison cloud depuis le réseau produit

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection license.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 license.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient l’établissement DNS/TCP depuis l’origine sélectionnée. Un succès ne prouve ni le proxy, [TLS](/kb/tls), le token, l’attribution de compte ni la transaction de licence. Les logs produit et le statut de l’éditeur fournissent la transition d’état suivante.

### Protéger un export d’audit contre les modifications

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\evidence\\license-inventory.csv -Algorithm SHA256
Get-FileHash .\\evidence\\entitlements.pdf -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./evidence/license-inventory.csv
sha256sum ./evidence/entitlements.pdf</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) détectent les modifications ultérieures d’octets. Il convient également de documenter l’heure de création, le périmètre de requête, la version de l’outil/de l’API, la requête, le fuseau horaire, l’exportateur et le stockage sécurisé. Un hash ne confirme pas l’exhaustivité métier.

### Vérifier les événements de licence et d’activation sur la période

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-01T00:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=$since} |
  Where-Object Message -Match 'licen[cs]e|activation|entitlement' |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-01 00:00:00 UTC' \
  --output short-iso-precise --no-pager \
  | grep -Ei 'licen[cs]e|activation|entitlement'</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) et [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrent les événements locaux. Les logs de l’éditeur peuvent utiliser des codes structurés au lieu de messages textuels en anglais ; les fournisseurs, Event ID ou champs définis sont à privilégier. L’heure, la rotation et la copie centralisée des logs font partie du constat.

L’entitlement, le point de comptage et l’état d’exploitation constituent la preuve d’audit. Celle-ci doit montrer de manière reproductible ce qui a été acheté, attribué, installé et effectivement utilisé.

## Preuve d’audit et d’exploitation

Une preuve de licences reproductible contient :

- référence du contrat/de la commande, produit/SKU, métrique, quantité, périmètre, durée et droits particuliers ;
- inventaire logiciel et matériel avec ID immuables, édition, version et relation de cluster ;
- attributions directes, par groupe et automatiques avec source et état de provisionnement ;
- valeurs de mesure brutes et règle de calcul pour chaque fenêtre temporelle ;
- exceptions avec justification, propriétaire, date d’expiration et contrôle technique ;
- HA/DR/test/migration comme population distincte ;
- affichage produit, export API/CLI et contre-calcul indépendant ;
- heure, fuseau horaire, version de requête/d’outil, hash et stockage protégé.

Les écarts sont classifiés : **erreur de données** (doublons, anciens comptes), **erreur de modèle** (métrique erronée), **erreur de processus** (licence non supprimée après un départ), **erreur technique** (synchronisation/token/serveur) ou **entitlement manquant**. Cette séparation indique seulement si la réponse est un nettoyage, une configuration, une clarification contractuelle, un achat ou une réponse à incident.

## Évolution technique des licences

Les Product Keys et dongles locaux ont d’abord lié étroitement le droit d’utilisation à un ordinateur. Les serveurs de licences réseau ont introduit des pools partagés et des leases simultanés ; la virtualisation a exigé de nouvelles règles d’hôtes, de cœurs et de mobilité. Les abonnements et le SaaS ont déplacé les entitlements vers des comptes cloud, groupes d’identité et plans de service. Cisco Smart Licensing et Microsoft Graph illustrent des modèles centralisés de comptes/API, tandis qu’ISO/IEC 19770 standardise les données SWID et d’entitlement ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallèlement, Open Source a rendu l’activation technique superflue pour de nombreux composants, mais non les conditions de licence. SPDX a créé des identifiants courts et des expressions lisibles par machine pour les chaînes logicielles. La tâche de l’administrateur est ainsi passée de « saisir une clé » à un problème de données portant sur le contrat, l’identité, l’actif, la durée, la télémétrie et la composition logicielle. La réponse est la même que pour la supervision : ancres d’objet claires, fenêtres temporelles explicites, données brutes et calculs reproductibles.

## Sources

- [ISO/IEC 19770-3:2016 – Schéma d’entitlement](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – Systèmes de gestion des actifs IT](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Software Identification Tag](https://www.iso.org/standard/65666.html)
- [Microsoft Graph – Liste des licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)
- [Cisco – Licences](https://www.cisco.com/site/us/en/buy/licensing/index.html)
- [Microsoft Graph PowerShell – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)
- [Microsoft – Éditions et versions d’Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions)
- [Microsoft – Saisir une Product Key Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)
- [Open Source Initiative – Open Source Definition](https://opensource.org/osd)
- [Apache Software Foundation – Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [SPDX Specification 3.0.1 – Expressions de licence](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)
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
