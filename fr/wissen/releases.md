---
title: "Versions et mises à jour : artefacts, déploiement et preuve d’exploitation"
blatt: "releases"
description: "Exploitation des versions et mises à jour pour les systèmes de messagerie et d’infrastructure : versions, canaux, cycle de vie, intégrité des artefacts, priorisation des vulnérabilités, dépendances, déploiement progressif, migration d’état, retour arrière, vérification et évolution technique."
fakten:
  - label: Processus
    wert: identifier · prioriser · obtenir · installer · vérifier l’installation
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Unité de version
    wert: artefact + version + plateforme cible + métadonnées + preuve d’intégrité + dépendances
    href: https://theupdateframework.io/specification/latest/
  - label: Version
    wert: à interpréter uniquement dans le modèle de versionnement documenté ; SemVer n’est pas universel
    href: https://semver.org/
  - label: Priorisation
    wert: exploitation · exposition · impact · mesure disponible · capacité de récupération
    href: https://www.cisa.gov/known-exploited-vulnerabilities-catalog
  - label: CVSS
    wert: gravité technique et métriques contextuelles ; ne constitue pas seul une échéance de correctif
    href: https://www.first.org/cvss/v4.0/specification-document
  - label: EPSS
    wert: probabilité statistique d’exploitation observée ; aucune évaluation des dommages
    href: https://www.first.org/epss/how-it-works.html
  - label: Intégrité
    wert: vérifier l’origine, le hash, la signature et, le cas échéant, la provenance de build
    href: https://slsa.dev/spec/v1.2/
  - label: Déploiement
    wert: pilote/canari → petites vagues → vagues étendues ; chaque vague dispose de critères d’arrêt
    href: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
  - label: Retour arrière
    wert: planifier séparément le retour du code, des données et de la configuration
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Validation de la messagerie
    wert: entrant · sortant · soumission · authentification · file d’attente · politique · stockage · supervision
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Limite de support
    wert: évaluer conjointement le produit, la branche, le niveau de correctif, la plateforme et les dépendances
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq
  - label: Preuve d’exploitation
    wert: version cible/réelle, ID d’artefact, changement, heure, résultat, écart et responsable
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - releases
  - patch-management
  - messaging
translationSourceHash: f5adb284b43cb4bb0db285f310beabb3b7e8b6ac825ea0abf603e08062c0cbbb
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:42:18.296Z
translationReview: automatic
---

# Versions et mises à jour : artefacts, déploiement et preuve d’exploitation

Une version est une unité nommée et publiée de logiciel ou de firmware ; une mise à jour est la transition contrôlée d’un système exploité d’un état approuvé vers un autre. Pour les administrateurs, cela comprend bien davantage que le fichier d’installation : plateforme cible, relation entre version et support, dépendances, migrations de configuration et de données, signature ou autre preuve d’origine, problèmes connus, chemin d’installation, critères d’arrêt et vérification fonctionnelle. Le NIST définit en conséquence la gestion des correctifs en entreprise comme l’identification, la priorisation, l’acquisition, l’installation et la vérification des correctifs, mises à jour et montées de version ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Les systèmes de messagerie accentuent cette tâche. Les écouteurs Internet traitent des entrées externes, les files d’attente portent des responsabilités non encore achevées, les clusters répartissent l’état et les utilisateurs attendent une distribution ininterrompue. Un programme d’installation réussi ne prouve donc ni un flux de messagerie opérationnel ni une pile globale prise en charge. L’exploitation des versions est une modification d’état dans une architecture de dépendances, avec une base de départ mesurable, une propagation contrôlée et un état final vérifiable.

L’explication commence par l’artefact concret et son origine. Viennent ensuite le type de version, l’évaluation des risques, les tests et le déploiement progressif ; l’arrêt, le retour arrière et la vérification clôturent le parcours de modification.

Une mise à jour est le passage contrôlé d’un état logiciel connu à un nouvel artefact. Le processus sûr commence par l’origine et la signature, passe par la qualification et un déploiement progressif, et ne s’achève qu’avec une réception technique et fonctionnelle.

## Objet de version et chaîne de confiance

Une validation robuste décrit l’**objet de version exact**. Le nom marketing « Version 16 » ne suffit pas lorsque l’édition, le build, le niveau de correctif, la plateforme, l’architecture ou le digest d’image déterminent le binaire réellement exécuté. Un nom de fichier n’est pas non plus une identité : dans un modèle de versionnement rigoureux, un artefact déjà publié ne doit pas être remplacé silencieusement. Semantic Versioning exige par exemple que le contenu d’une version publiée demeure inchangé ; cette règle ne s’applique toutefois que si le fabricant garantit réellement SemVer ([Semantic Versioning 2.0.0](https://semver.org/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-releases.svg?v=20260813" title="Interaktive Infografik: Releaseobjekt, Vertrauenskette, Risikopriorisierung, Betriebsmodelle, gestufter Rollout, Abbruch, Rollback und Verifikation" loading="lazy">
  <a href="/images/kb-interaktiv-releases.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

Un objet de version comprend au minimum :

| Élément | Question d’administration | Preuve |
|---|---|---|
| Artefact | Quels octets sont installés ou démarrés ? | Hash, ID de paquet, digest d’image ou identifiant de firmware |
| Fabricant et origine | De quel canal approuvé provient-il ? | Dépôt HTTPS, métadonnées signées, portail du fabricant, chaîne de transfert hors ligne |
| Version et cible | Pour quel produit, édition, système d’exploitation, architecture et version de départ ? | Notes de version, matrice de compatibilité, chemin de mise à niveau |
| Intégrité et authenticité | L’artefact est-il inchangé et autorisé par l’éditeur attendu ? | Signature, vérification du certificat/de la clé, hash issu d’une source authentifiée |
| Contenu et provenance | Quels composants et quel build l’ont produit ? | SBOM, provenance de build, attestation, identité du système de build |
| Modification d’état | Quels schémas, index, files d’attente, configurations ou clés sont modifiés ? | Plan de migration, étapes irréversibles, besoins de stockage et durée |
| Contrat d’exploitation | Quels services, ports, API, paramètres par défaut et dépendances changent de comportement ? | Changements incompatibles, dépréciations, problèmes connus, nouveaux niveaux minimums |

The Update Framework (TUF) modélise une distribution sécurisée des mises à jour avec des rôles signés pour les métadonnées Root, Targets, Snapshot et Timestamp. Il vise notamment à limiter les clés de dépôt compromises, le retour à d’anciens états et les métadonnées figées. SLSA décrit la provenance de build comme une information vérifiable sur la manière, le lieu et les entrées ayant produit un artefact. Ces deux approches montrent qu’un hash ne confirme que l’égalité avec une valeur de référence ; seule l’origine fiable de cette valeur répond à la question de l’authenticité ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

## Pile technologique du processus de version

L’exploitation des versions est elle-même une pile technologique. Elle relie le système source et de build, le format d’artefact ou de paquet, les procédures de signature et de provenance, le dépôt ou registre, le canal de distribution, l’installateur ou l’orchestrateur, la migration de configuration et de schéma, ainsi que la télémétrie et la gestion du retour arrière. TUF décrit les rôles de métadonnées signées d’un dépôt ; SLSA décrit la provenance entre l’état source, la plateforme de build et l’artefact ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

Pour un produit cible, sa propre pile s’y ajoute : système d’exploitation, architecture CPU, runtime, base de données, annuaire, proxy, extensions et clients. Une validation n’est qualifiée que si ces deux piles sont compatibles. Une image de conteneur signée valide peut être inutilisable sur la mauvaise architecture ; un gestionnaire de paquets ayant réussi ne peut pas pour autant annuler une migration de base de données non prise en charge.

## Versions, canaux et limites de support

Les numéros de version sont des noms relevant d’un contrat de fabricant, et non une échelle universelle de risques. Avec SemVer, `MAJOR.MINOR.PATCH` désignent respectivement une modification d’API incompatible, une fonctionnalité rétrocompatible et une correction de bogue rétrocompatible. Les fabricants peuvent toutefois utiliser des modèles calendaires, de build, de firmware, de mises à jour cumulatives, de fonctionnalités ou d’abonnement. Sans documentation, `3.9.1` n’implique ni que `3.10.0` soit compatible, ni qu’un correctif soit mineur ou peu risqué ([Semantic Versioning 2.0.0](https://semver.org/)).

Les termes « stable », « LTS », « extended », « preview », « early access » et « general availability » sont également des canaux de version propres à chaque fabricant. Un inventaire d’administration enregistre donc non seulement la version en cours, mais aussi le canal, la fin du support, les chemins prédécesseur/successeur pris en charge et la matrice de plateformes. L’élément déterminant est l’**ensemble de support** composé de la branche produit, du niveau de correctif, du système d’exploitation, du runtime, de la base de données, de l’annuaire, du navigateur/client, des extensions et, le cas échéant, du modèle matériel.

Le modèle de maintenance d’Exchange de Microsoft illustre cette multidimensionnalité : les mises à jour cumulatives, les mises à jour de sécurité et les mises à jour correctives ont des prérequis et relations différents ; les mises à jour de sécurité sont spécifiques aux CU, et des mesures manuelles supplémentaires peuvent être nécessaires. La règle statique d’administration n’est donc pas « Exchange reçoit des correctifs chaque mois », mais : pour chaque serveur, vérifier conjointement le niveau CU/SU pris en charge, l’état de Windows, les Management Tools et les actions supplémentaires associées à l’avis concret ([Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)).

Une fois l’artefact, le canal et le statut de support établis, le risque de modification est évalué. Un numéro de version élevé ne dit à lui seul ni l’urgence ni la compatibilité.

## Priorisation : le risque plutôt que le numéro de version

Toute nouvelle version n’est pas urgente, et toute vulnérabilité critique ne doit pas attendre la prochaine fenêtre de routine. La priorisation combine plusieurs signaux indépendants :

1. **Preuve d’exploitation :** CISA tient le Known Exploited Vulnerabilities Catalog comme liste faisant autorité des vulnérabilités activement exploitées et recommande son utilisation comme entrée de priorisation ([CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)).
2. **Gravité technique :** CVSS v4.0 décrit les métriques de base, de menace, environnementales et supplémentaires. Le score ne constitue pas une déclaration complète sur son propre actif ou sur une échéance de correctif ([FIRST – CVSS v4.0](https://www.first.org/cvss/v4.0/specification-document)).
3. **Probabilité d’exploitation :** EPSS estime la probabilité qu’une CVE publiée soit effectivement exploitée dans une fenêtre de temps. EPSS n’évalue pas les dommages potentiels et ne remplace pas la connaissance des actifs ([FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)).
4. **Exposition propre :** Les écouteurs Internet, une interface de gestion accessible, l’authentification existante, une fonction active, la version, les contrôles compensatoires et la valeur des données déterminent si le chemin décrit existe dans son propre système.
5. **Impact opérationnel :** Une panne, la croissance des files d’attente, une migration de données, la perte de cluster ou des dépendances non testées peuvent être causées par la mise à jour elle-même.
6. **Capacité de récupération :** Un retour arrière testé, des nœuds de remplacement et un [modèle de sauvegarde/DR](/kb/backup-dr) à jour permettent une vitesse différente d’une mise à niveau de schéma irréversible sans test de restauration.

Le résultat est une échéance justifiée avec un responsable, et non simplement « CVSS 9.8 ». Une vulnérabilité activement exploitée sur une passerelle Internet peut exiger des mesures immédiates, une réduction de l’exposition et une fenêtre d’urgence. Un composant local inutilisé sur un système isolé peut être planifié dans une vague régulière avec une compensation documentée. Les exceptions reçoivent une date d’expiration ; sinon elles deviennent des écarts permanents invisibles.

## Approches architecturales des mises à jour

Le retour arrière et le périmètre de panne dépendent de l’approche architecturale :

| Modèle d’exploitation | Modification d’état | Modèle de déploiement | Limite centrale |
|---|---|---|---|
| Serveur ou appliance in situ | Binaires, paquets et souvent configuration sur le même nœud | Mode maintenance, mise à jour du nœud, vérification, reprise | une mise à jour partielle ou un installateur défaillant peut laisser le nœud dans un état incertain |
| Cluster avec rôles à état | Nœud plus état de réplication/quorum | Déplacer les rôles, isoler un nœud, mettre à jour, resynchroniser, puis vague suivante | exploitation mixte des versions, quorum, charge de reconstruction et ordre des versions |
| Image immuable / conteneur | une nouvelle image remplace l’ancienne instance ; configuration et données sont externes | Canari, RollingUpdate, Blue/Green | le retour arrière de l’image ne résout pas une migration de données irréversible |
| SaaS / service géré | le fabricant déploie le service ; le client contrôle la configuration et les intégrations | vagues annoncées, Feature Flags, compatibilité tenant/API | aucun retour arrière binaire par le client ; contrat, télémétrie et test d’intégration sont essentiels |
| Firmware / appliance matérielle | firmware, partitions de démarrage, pilotes et éventuellement format de données | nœud en veille/HA d’abord ou double banque, puis côté actif | capacité de démarrage, modèle matériel et chemin de rétrogradation dépendent du fabricant |

Les Deployments Kubernetes montrent explicitement le modèle roulant : `maxUnavailable` et `maxSurge` limitent respectivement les Pods indisponibles et supplémentaires pendant le remplacement ; le déploiement peut être observé, suspendu et ramené à une révision enregistrée. Cela ne garantit que l’état du contrôleur. Une migration de base de données, un état de file d’attente externe ou des clients incompatibles restent hors de ce retour arrière ([Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)).

Pour les clusters de messagerie, « nœud en veille d’abord » n’est également correct que si le fabricant prend en charge ce chemin. Les clusters de configuration pair à pair, les pools d’équilibrage de charge, la réplication de bases de données et la haute disponibilité des files d’attente sont des mécanismes différents. Avant la mise à jour, il doit être clair quels états sont répliqués, quelles versions peuvent collaborer temporairement et si un nœud mis à jour écrit les données dans un format que l’ancien nœud peut encore lire.

## Les notes de version comme contrat technique

Les notes de version ne sont pas parcourues à la recherche de mots-clés, mais traduites dans une matrice locale de modifications :

- **Sécurité :** CVE, composant/fonction affecté, vecteur d’attaque, privilège, exploitation connue, mesure d’atténuation et action supplémentaire.
- **Prérequis :** versions de départ autorisées, versions intermédiaires, espace libre, runtime, système d’exploitation, schéma, licence et modèle matériel.
- **Comportement :** nouveaux paramètres par défaut, ports modifiés, chiffrements/protocoles, champs API/CLI, authentification, certificats, routage du courrier et limites.
- **Suppression :** dépréciations, fonctionnalités supprimées, anciens algorithmes, plateformes ou interfaces de gestion abandonnées.
- **Migration :** durée, verrous, construction de file d’attente/d’index, format de données, rétrocompatibilité et travaux ultérieurs requis.
- **Problèmes connus :** déclencheur, détection, solution de contournement, objectif de correction et effet sur le retour arrière ou la HA.
- **Chemin d’installation :** ordre, redémarrages, rôles de nœuds, contrôles de santé et étapes manuelles ultérieures.

La documentation du fabricant est la source primaire ; les forums et retours d’expérience peuvent fournir des hypothèses, mais ne remplacent pas une relation de mise à niveau prise en charge. Le NIST SSDF exige d’archiver les logiciels publiés et leurs composants, de protéger leur intégrité et de fournir des informations sur la correction des vulnérabilités. Pour les exploitants, il en résulte que les notes de version, artefacts, informations de hash/signature, décision de validation et résultats de tests locaux sont conservés ensemble de manière référencable ([NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

Les notes de version décrivent les modifications attendues ; la qualification les vérifie par rapport à sa propre topologie, au volume de données, aux intégrations et aux possibilités de repli.

## Qualification avant la fenêtre de maintenance

L’environnement de test ne doit pas avoir la même taille, mais doit être représentatif dans les **propriétés pertinentes pour le risque** : même édition de produit, authentification, chaînes de certificats, version d’annuaire et de base de données, règles de flux de messagerie, extensions, tailles de messages, protocoles et chemin de mise à niveau. Un laboratoire fraîchement installé ne teste pas une erreur de mise à niveau qui ne survient qu’avec une configuration développée au fil des années.

Avant la validation, il faut au minimum vérifier :

1. le téléchargement depuis une source approuvée et la preuve d’intégrité/de signature ;
2. les prérequis et le chemin intermédiaire documenté ;
3. la sauvegarde de configuration et de données avec un chemin de restauration connu ;
4. la mise à niveau d’une copie représentative ou d’un nœud pilote ;
5. les transactions fonctionnelles, y compris les cas négatifs et les décisions de politique ;
6. les performances, la file d’attente, la réplication, l’index/reconstruction et la croissance du stockage ;
7. l’effet de la supervision et des alertes pendant la panne, la dégradation et le retour ;
8. le retour arrière ou la restauration jusqu’au point de décision défini ;
9. la durée avec une réserve pour l’arrêt, la récupération et la vérification.

Un instantané n’est pas un retour arrière universel. Dans les systèmes distribués, la base de données, la file d’attente, le Blob Store, l’annuaire, les certificats et les pairs externes peuvent continuer après l’instantané. Réinitialiser un seul nœud peut créer un état incohérent ou déjà retraité. Le [modèle de sauvegarde et de DR](/kb/backup-dr) doit désigner le point de cohérence commun et les limites de responsabilité.

## Déploiement progressif et critères d’arrêt

Un déploiement contrôlé limite le rayon d’impact par vagues :

1. **Pilote/Canari :** unité petite, observable et représentative, sans fonction unique.
2. **Vague précoce :** plusieurs systèmes ou utilisateurs afin de révéler des dépendances plus rares.
3. **Vagues étendues :** capacité, régions, rôles ou tenants dans des lots définis.
4. **Observation postérieure :** au moins sur les cycles pertinents de relance, cache, file d’attente, certificats et charge quotidienne.

Chaque vague a des **critères Go, Hold et Abort** définis à l’avance : taux d’erreur, latence, âge/profondeur de file d’attente, répartition SMTP 4xx/5xx, erreurs d’authentification, retard de réplication, ressources, boucles de crash, intégrité des données et signalements au support. « Aucun ticket » n’est pas un contrôle de santé suffisant. La vague suivante ne débute que lorsque la période de mesure et les critères sont remplis.

En cas d’arrêt, le retour arrière n’est pas automatique. Si un retour arrière est plus dangereux que le maintien — par exemple après une modification de schéma orientée vers l’avant —, la réaction sûre peut être de suspendre le déploiement, retirer le trafic du nœud concerné, sauvegarder les données et suivre le chemin du fabricant. La matrice de décision est établie avant la fenêtre, et non improvisée sous pression.

Après une simple modification binaire, un retour arrière peut être simple. Dès que le schéma, l’index ou les données enregistrées ont changé, il faut déterminer au préalable si un retour est encore techniquement possible.

## Retour arrière, progression et migration de données

« Revenir à l’ancienne version » se décompose en au moins quatre questions de retour indépendantes :

- **Code :** L’ancien paquet, image ou image de firmware est-il disponible et signé ? Le fabricant prend-il en charge la rétrogradation ?
- **Configuration :** Le format a-t-il changé, et l’ancien état comprend-il la nouvelle configuration ?
- **Données :** Un schéma ou un index a-t-il été migré vers l’avant ? L’ancien logiciel peut-il lire ces données, ou une restauration est-elle nécessaire ?
- **Effet externe :** Des messages ont-ils été acceptés, des jetons émis, le DNS modifié, des certificats renouvelés ou des API utilisées par d’autres systèmes ?

Pour les frontaux sans état, un retour arrière d’image est souvent rapide. Sur une plateforme de messagerie, la même opération peut traiter les messages deux fois ou perdre des états nouvellement acceptés. Il existe donc un **point de non-retour** : jusqu’à celui-ci, un retour arrière technique est testé ; ensuite, une progression, une restauration ou un mode de compatibilité défini est prévu.

Le retour arrière ne remplace pas non plus l’analyse des causes. Après un recul, un artefact défectueux, un prérequis incomplet, un écart local, une limite de capacité et un défaut de produit inconnu restent des hypothèses distinctes. Le constat de version enregistre la version exacte, l’hôte, la phase, le code d’erreur, l’heure et les journaux sécurisés, puis suit une [méthodologie de dépannage fondée sur les preuves](/kb/troubleshooting).

## Outils d’administration pour l’inventaire et la vérification

Les commandes fournissent des éléments pour justifier un changement ; elles ne remplacent ni les contrôleurs de santé du fabricant ni les instructions de mise à niveau spécifiques au produit. La sortie est enregistrée avant et après la mise à jour avec l’heure, l’hôte et l’ID de changement. Les secrets ou configurations complètes ne doivent pas être placés sans protection dans les tickets.

### Relever l’inventaire de plateforme et de correctifs

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ComputerInfo -Property WindowsProductName,WindowsVersion,OsBuildNumber,OsArchitecture
Get-HotFix | Sort-Object InstalledOn,HotFixID</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">uname -a
cat /etc/os-release
rpm -qa --qf '%{NAME}\t%{EPOCHNUM}:%{VERSION}-%{RELEASE}\t%{ARCH}\n' 2&gt;/dev/null \
  || dpkg-query -W -f='${binary:Package}\t${Version}\n'</code></pre>
  </div>
</div>

[`Get-ComputerInfo`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo) et [`Get-HotFix`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix) recensent les informations Windows et les mises à jour enregistrées. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html), [`os-release`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html), [`rpm`](https://rpm.org/docs/6.0.x/man/rpm.8) et [`dpkg-query`](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html) fournissent les données du système d’exploitation et des paquets sous Linux. Le build produit, le firmware de l’appliance, le digest de conteneur et les extensions doivent aussi être recensés avec les outils respectifs des fabricants.

### Vérifier le hash et la signature de l’artefact

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash C:\\Staging\\update.msi -Algorithm SHA256
Get-AuthenticodeSignature C:\\Staging\\update.msi |
  Format-List Status,StatusMessage,SignerCertificate,TimeStamperCertificate</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum update.tar.gz
gpgv release-signature.asc release-manifest.txt</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparent les octets à une valeur de référence authentifiée. [`Get-AuthenticodeSignature`](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature) et [`gpgv`](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html) vérifient les signatures par rapport à l’ancre de confiance configurée. Un statut « valide » ne répond à la question de l’éditeur que si le certificat, la clé, le nom, la période de validité et le chemin de référence sont attendus.

### Comparer les services et écouteurs avant et après le déploiement

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Dienst- und Listenerprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Service | Sort-Object Name | Select-Object Name,Status,StartType
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort | Select-Object LocalAddress,LocalPort,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">systemctl --failed --no-pager
systemctl list-units --type=service --state=running --no-pager
ss -lntup</code></pre>
  </div>
</div>

[`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service), [`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection), [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) montrent l’état local des services et écouteurs. Un processus en cours d’exécution n’est pas encore un contrôle de santé fonctionnel ; l’adresse de liaison, le port, le processus, TLS, la réponse de protocole et les dépendances en aval sont vérifiés séparément.

### Vérifier l’accessibilité d’un point de terminaison représentatif

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Smoke-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection admin.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
nc -vz -w 5 admin.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) démontrent l’établissement TCP depuis l’origine choisie. Viennent ensuite les tests [TLS](/kb/tls)-, [SMTP](/kb/smtp)-, d’authentification et fonctionnels du flux de messagerie ; un port ouvert ne confirme pas un service opérationnel.

### Conserver les erreurs et avertissements pendant la fenêtre de changement

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T18:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since; Level=1,2,3} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 18:00:00 UTC' \
  --priority warning --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) et [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) limitent le constat à la fenêtre de maintenance. Les journaux de l’installateur, du produit, du cluster et de la base de données sont également conservés avec la source exacte et la référence temporelle. La vérification ne recherche pas seulement de nouvelles erreurs, mais aussi des événements attendus absents, des redémarrages répétés et des nouvelles tentatives croissantes.

Un état de processus sans anomalie ne suffit pas pour la messagerie. L’acceptation, la file d’attente, la politique, la distribution sortante, l’accès client et le suivi doivent être vérifiés sur le chemin réel des messages.

## Validation spécifique à la messagerie

Après la mise à jour, on ne se contente pas d’envoyer un courriel de test. La validation couvre les rôles de la plateforme :

- **Relais entrant :** sélection DNS/MX externe, TCP/TLS, vérification du destinataire, acceptation après DATA, politique de contenu et de domaine, file d’attente et distribution finale.
- **Relais sortant :** sélection du connecteur/Smarthost, IP source, politique TLS, [SPF/DKIM/DMARC](/kb/mail-auth), nouvelle tentative de file d’attente et acceptation par un destinataire contrôlé.
- **Soumission et accès :** authentification, autorisation, IMAP/HTTP/API, Send-as/Delegate et clients mobiles/anciens dans la mesure prise en charge.
- **Identité :** liaison et recherche [LDAP](/kb/ldap), résolution des groupes/attributs, [Kerberos](/kb/kerberos), jetons et certificats.
- **Politique et cryptographie :** malware/DLP, quarantaine, avertissement, routage, [S/MIME/OpenPGP/Portal](/kb/verschluesselung) et accès aux clés.
- **État et HA :** file d’attente, base de données, stockage d’objets, réplication, index, agent de sauvegarde, basculement et retour du nœud mis à jour.
- **Administration :** GUI, API, CLI, rôles, journal d’audit, export de configuration et supervision.

Chaque test possède une réponse attendue et réelle, un ID de corrélation, une heure, une origine et une destination. Un contrôle synthétique vert et un petit pilote réel se complètent : le contrôle synthétique est reproductible ; le pilote révèle des combinaisons inattendues de clients, politiques et charges. La séparation entre transport, authentification de domaine et mécanismes de contenu suit l’architecture technique du courriel dans NIST SP 800-177 Rev. 1 ([NIST – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)).

## Évolution technique de l’exploitation des versions

Autrefois, les états système étaient souvent distribués comme supports complets ou correctifs du fabricant appliqués manuellement. Les gestionnaires de paquets, dépôts signés et modèles de maintenance cumulatifs ont automatisé les dépendances et la distribution ; les machines virtuelles, images et conteneurs ont déplacé l’accent vers le remplacement reproductible des instances. CI/CD a réduit le temps entre le build et la production, tandis que TUF, SLSA et SSDF modélisent explicitement la chaîne de confiance du code source, du système de build, des métadonnées et de l’artefact ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/), [NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

La conséquence opérationnelle n’est pas « tout installer automatiquement », mais **identifier chaque modification d’état de façon lisible par machine, la qualifier de manière reproductible, la déployer de façon limitée et la vérifier fonctionnellement**. L’automatisation réduit les écarts manuels ; sans vagues ni logique d’arrêt, elle augmente simultanément la vitesse d’une erreur. Les plateformes modernes de versions nécessitent donc les mêmes contrôles classiques — inventaire, validation, fenêtre de maintenance et récupération — complétés par la provenance de l’artefact, l’état cible déclaratif, la télémétrie et la distribution progressive.

## Sources

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [NIST SP 800-177 Rev. 1 – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [The Update Framework – Specification](https://theupdateframework.io/specification/latest/)
- [SLSA – Specification 1.2](https://slsa.dev/spec/v1.2/)
- [Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)
- [CISA – Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [FIRST – CVSS v4.0 Specification](https://www.first.org/cvss/v4.0/specification-document)
- [FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)
- [Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [NIST – SP 800-218, Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [Microsoft Learn – Get-ComputerInfo](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo)
- [Microsoft Learn – Get-HotFix](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix)
- [GNU Coreutils – uname](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html)
- [systemd – os-release](https://www.freedesktop.org/software/systemd/man/latest/os-release.html)
- [RPM – rpm manpage](https://rpm.org/docs/6.0.x/man/rpm.8)
- [Debian – dpkg-query](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-AuthenticodeSignature](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [GnuPG – gpgv](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
