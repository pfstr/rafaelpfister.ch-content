---
title: "Conteneurs : images, exécution et exploitation robuste"
blatt: "container"
description: "Conteneurs pour administrateurs d’infrastructure et de messagerie : artefacts OCI, isolation par runtime et noyau, espaces de noms et cgroups, réseau et stockage, orchestration, identité et chaîne d’approvisionnement, ressources, santé, journalisation, reprise, conteneurs Windows et histoire technique."
fakten:
  - label: Rôle système
    wert: ensemble de processus isolés avec système de fichiers racine empaqueté et limites explicites de ressources et d’E/S
    href: https://csrc.nist.gov/pubs/sp/800/190/final
  - label: Standard d’artefact
    wert: OCI Image Specification · manifeste · configuration · couches · descripteurs
    href: https://specs.opencontainers.org/image-spec/
  - label: Standard d’exécution
    wert: OCI Runtime Specification · bundle · config.json · cycle de vie
    href: https://specs.opencontainers.org/runtime-spec/
  - label: Base Linux
    wert: espaces de noms · cgroups · capabilities · LSM · Seccomp
    href: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - label: Base Windows
    wert: isolation Process ou Hyper-V avec images de conteneurs Windows
    href: https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/
  - label: Persistance
    wert: volume, Bind Mount ou service externe ; la couche inscriptible n’est pas une stratégie de reprise
    href: https://docs.docker.com/engine/storage/
  - label: Réseau
    wert: espace de noms réseau/vNIC · bridge/overlay · publication de ports · DNS/découverte de services
    href: https://github.com/containernetworking/cni/blob/main/SPEC.md
  - label: Ressources
    wert: CPU · mémoire · PID · E/S ; considérer séparément limites et réservations
    href: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - label: Orchestration
    wert: état souhaité, planification, redémarrage, déploiement et découverte de services ; ne fait pas partie de l’image
    href: https://kubernetes.io/docs/concepts/workloads/pods/
  - label: Chaîne d’approvisionnement
    wert: digest · registre · provenance/attestation · signature · politique
    href: https://slsa.dev/spec/v1.2/
  - label: État d’exploitation
    wert: processus · santé · ressources · réseau · stockage · journaux · événements · état souhaité
    href: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
  - label: Objet de reprise
    wert: définition de déploiement, images/digests, configuration, secrets/clés et données persistantes
    href: https://csrc.nist.gov/pubs/sp/800/190/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - paperless-ngx
  - home-assistant
translationSourceHash: 7091efc5400536d4b1b40e368e564d2ab0d5332766f61e0fdaf976e723bff142
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:27:23.981Z
translationReview: required
---

# Conteneurs : images, exécution et exploitation robuste

Un conteneur n’est pas un petit serveur, et une image n’est pas une sauvegarde. Techniquement, un runtime de conteneur exécute des processus ordinaires avec un système de fichiers préparé, des vues isolées des ressources du noyau et des limites de ressources et de sécurité définies. Le noyau hôte reste impliqué dans chaque opération. Cette architecture rend les environnements applicatifs reproductibles et rapidement remplaçables ; elle déplace toutefois la responsabilité vers la provenance des images, le runtime, le réseau, la persistance, les secrets, le contrôle des ressources et l’orchestration.

Pour les administrateurs, la distinction la plus importante est celle entre **artefact**, **instance d’exécution** et **état d’exploitation**. Une image décrit le système de fichiers démarrable et les métadonnées. Un conteneur est une exécution concrète. Les bases de données, files d’attente, clés, configurations et données d’audit ont leurs propres cycles de vie. Quiconque traite conjointement ces niveaux comme « le conteneur Docker » ne peut ni circonscrire un incident ni planifier entièrement une restauration.

L’explication commence par l’image en tant qu’artefact livrable et suit son parcours via le moteur et le runtime jusqu’au noyau, au réseau et au stockage. L’orchestration, la sécurité, le diagnostic et la reprise s’appuient sur ce déroulement.

## Positionnement technique

NIST décrit les conteneurs d’application comme une forme de virtualisation du système d’exploitation : les applications partagent le noyau, tandis que l’isolation et le contrôle des ressources reposent sur des mécanismes du système d’exploitation. Les machines virtuelles virtualisent en revanche le matériel pour leur propre noyau invité. Les conteneurs et les VM peuvent être combinés ; le choix modifie la surface d’attaque, la compatibilité, le coût de démarrage et le périmètre de défaillance ([NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)).

Un chemin Linux typique se compose de :

1. registre ou magasin de contenu local avec manifestes, configurations et couches OCI ;
2. moteur ou orchestrateur qui définit l’état souhaité, le réseau, les montages et les politiques ;
3. runtime de haut niveau tel que `containerd` ou CRI-O pour la gestion des images et conteneurs ;
4. runtime OCI tel que `runc`, qui crée le processus à partir du bundle et de `config.json` ;
5. noyau hôte avec espaces de noms, cgroups, capabilities, Seccomp et un Linux Security Module ;
6. processus de conteneur qui interagissent avec leur environnement via des systèmes de fichiers virtuels ou montés, des sockets et des périphériques.

L’Open Container Initiative standardise séparément les parties image, runtime et distribution. Ainsi, une image OCI peut être traitée par différents moteurs et runtimes sans que le réseau, l’orchestration ou la sauvegarde des données soient automatiquement standardisés ([OCI Image Specification](https://specs.opencontainers.org/image-spec/), [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/), [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-container.svg?v=20260813" title="Interaktive Infografik: Container von OCI-Artefakt und Registry über Engine, Runtime und Kernelisolation bis Netzwerk, Storage, Identität, Supply Chain und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-container.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Image : manifeste, configuration et couches

Une image OCI est un graphe de contenu orienté. Un **manifeste** référence, par des descripteurs et des digests cryptographiques, une configuration d’image et des couches de système de fichiers ordonnées. Un index d’image facultatif peut regrouper plusieurs manifestes pour différents systèmes d’exploitation et architectures. Le type de média, le digest et la taille font partie du descripteur ; les tags relèvent en revanche de la résolution de noms du registre et ne constituent pas une identité immuable ([OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md), [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)).

Les couches contiennent des modifications du système de fichiers. Au démarrage, elles sont assemblées par un pilote de stockage en une vue racine commune, complétée par une couche de conteneur inscriptible. Un secret ou un contenu de paquet supprimé peut continuer d’exister dans une couche antérieure. Les builds multi-étapes réduisent les outils de build et les artefacts intermédiaires dans l’image finale, mais ne remplacent pas la vérification des secrets et de la provenance ([Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/), [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)).

### Tags, digests et sélection de plateforme

Un tag tel que `stable`, `3` ou `latest` peut être déplacé vers un autre digest de manifeste. Une production reproductible épingle donc le digest approuvé ou documente au minimum le digest résolu lors du déploiement. Pour les images multi-architectures, le client sélectionne une plateforme dans l’index ; une architecture incorrecte, des fonctionnalités CPU manquantes ou l’émulation peuvent provoquer un comportement différent malgré un tag identique.

Un digest répond à « quels octets ? », non à « qui les a construits ? » ou « sont-ils sûrs ? ». Les signatures et attestations peuvent lier identité, provenance de build et matériaux. SLSA décrit la provenance comme une information vérifiable indiquant où, quand et comment un artefact a été produit ; `cosign` de Sigstore peut signer et vérifier des artefacts de conteneur ([SLSA Specification](https://slsa.dev/spec/v1.2/), [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Imageinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$image = $env:CONTAINER_IMAGE
docker buildx imagetools inspect $image
docker image inspect $image --format '{{json .RepoDigests}}'
cosign verify $image --certificate-identity $env:EXPECTED_IDENTITY `
  --certificate-oidc-issuer $env:EXPECTED_ISSUER</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">image="$CONTAINER_IMAGE"
docker buildx imagetools inspect "$image"
docker image inspect "$image" --format '{{json .RepoDigests}}' | jq .
cosign verify "$image" --certificate-identity "$EXPECTED_IDENTITY" \
  --certificate-oidc-issuer "$EXPECTED_ISSUER"</code></pre>
  </div>
</div>

[`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/) affiche les listes de manifestes et les plateformes, [`docker image inspect`](https://docs.docker.com/reference/cli/docker/image/inspect/) les métadonnées locales de l’image. [`cosign verify`](https://docs.sigstore.dev/cosign/verifying/verify/) vérifie la signature et l’identité attendue ; [`jq`](https://jqlang.org/manual/) formate le JSON dans l’exemple Unix.

À partir de l’image immuable, le runtime crée le conteneur réellement exécuté. Ce n’est qu’alors que la couche inscriptible, les espaces de noms, les cgroups, les montages et les paramètres du processus sont réunis.

## Runtime, bundle et cycle de vie du conteneur

L’OCI Runtime Specification décrit un conteneur comme un environnement pour un processus. Un **bundle** contient `config.json` et un système de fichiers racine. La configuration définit les arguments du processus, l’environnement, l’utilisateur, les montages, les espaces de noms, les ressources et d’autres options de plateforme. Le cycle de vie distingue la création, le démarrage, l’arrêt et la suppression ; un processus ayant le statut `created` ne s’exécute pas encore ([OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)).

`runc` est une implémentation de référence de cette interface bas niveau. `containerd` gère par-dessus les images, snapshots, conteneurs, tâches et événements. Docker Engine fournit une API de plus haut niveau avec réseau, volumes, build et modèle d’utilisation. Sur un nœud, Kubernetes communique avec un runtime compatible via la **Container Runtime Interface (CRI)** ; kubelet ne s’adresse pas simplement à la CLI Docker ([runc](https://github.com/opencontainers/runc), [containerd](https://containerd.io/docs/), [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)).

Ces couches possèdent des états distincts. Un pod Kubernetes peut signaler `Running` alors qu’un processus de conteneur se trouve dans une boucle de redémarrage ; un moteur peut connaître un conteneur dont la tâche runtime n’est plus active ; un processus peut être en cours d’exécution alors que le service n’accepte aucun trafic. Le diagnostic commence donc par la question : **état souhaité, objet moteur, tâche runtime, processus ou application ?**

## Espaces de noms : vue isolée, pas machine autonome

Les espaces de noms Linux isolent les ressources globales dans des vues distinctes. Les pages de manuel du noyau citent notamment les espaces de noms Mount, PID, Network, IPC, UTS, User, cgroup et Time. Un processus ne peut appartenir qu’à un seul espace de noms de chaque type ; les relations peuvent être examinées via `/proc/<pid>/ns` ([namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)).

| Espace de noms | Vue isolée | Pertinence administrative |
|---|---|---|
| Mount | Points de montage et système de fichiers racine | Bind Mounts, propagation, chemins hôte masqués |
| PID | Identifiants et hiérarchie de processus | PID 1, comportement des signaux et récupération des processus enfants |
| Network | Interfaces, routes, ports, état du pare-feu | Le conteneur peut écouter en interne sans être accessible de l’extérieur |
| UTS | Nom d’hôte et nom de domaine | Identité cosmétique, ni DNS ni principal de sécurité |
| IPC | IPC System V et files de messages POSIX | IPC partagé uniquement avec une configuration explicite |
| User | Mappage des UID/GID | Le root du conteneur peut être mappé comme non privilégié à l’extérieur |
| cgroup | Vue sur la hiérarchie cgroup | N’empêche pas à lui seul l’utilisation des ressources |
| Time | Décalages de temps monotone/de démarrage sélectionnés | Rarement utilisé ; aucune isolation générale de fuseau horaire |

L’isolation est configurable. Le réseau hôte, le PID hôte, `--privileged`, les autorisations de périphériques, les Bind Mounts étendus ou des capabilities supplémentaires ouvrent délibérément des frontières. Le nom du conteneur et un UID interne ne constituent pas un principal de sécurité à travers l’hôte, le registre et l’orchestrateur.

### PID 1, signaux et arrêt propre

Dans l’espace de noms PID, le premier processus assume des tâches particulières. Il reçoit les signaux différemment et doit récupérer les processus enfants orphelins. Les wrappers shell qui ne démarrent pas le programme réel avec `exec` peuvent absorber les signaux d’arrêt. Les orchestrateurs envoient généralement d’abord un signal de terminaison, attendent une période de grâce, puis forcent l’arrêt. L’application doit cesser d’accepter du nouveau travail, terminer les opérations en cours dans un délai limité et vider les états.

Docker peut utiliser un petit processus init avec `--init` ; cela ne corrige pas une application, mais rend plus explicites le relais des signaux et la récupération des processus enfants ([Docker run reference – `--init`](https://docs.docker.com/reference/cli/docker/container/run/#init)). Pour les files d’attente de courrier, les bases de données et les indexeurs, la durée maximale d’arrêt sûre doit faire partie du modèle de déploiement.

## cgroups : contrôle et comptabilisation

Les cgroups regroupent des processus et appliquent des contrôleurs aux ressources. Dans cgroup v2, les processus forment une hiérarchie unifiée. Les contrôleurs CPU, mémoire, E/S et PID ont des sémantiques différentes : une limite CPU ralentit, une limite mémoire peut déclencher un OOM kill, une limite PID empêche de nouveaux processus ou threads, et les limites d’E/S dépendent du chemin réel du périphérique de bloc ([Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)).

Il ne faut pas confondre **request/réservation** et **limit**. Kubernetes utilise les requests pour la planification et les limits pour les limites d’exécution ; pour la mémoire, la limite est réactive et appliquée par le noyau sous pression. Un pod peut être évincé ou arrêté malgré une capacité totale suffisante en raison d’une pression sur le nœud ou le cgroup ([Kubernetes – Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerprozess- und Ressourceninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker container inspect $env:CONTAINER_NAME --format '{{json .State}}'
docker container top $env:CONTAINER_NAME
docker stats --no-stream $env:CONTAINER_NAME
Get-Process -Name $env:HOST_PROCESS_NAME -ErrorAction SilentlyContinue</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker container inspect "$CONTAINER_NAME" --format '{{json .State}}' | jq .
docker container top "$CONTAINER_NAME"
docker stats --no-stream "$CONTAINER_NAME"
pid="$(docker inspect --format '{{.State.Pid}}' "$CONTAINER_NAME")"
ps -o pid,ppid,user,stat,args -p "$pid"</code></pre>
  </div>
</div>

[`docker container inspect`](https://docs.docker.com/reference/cli/docker/container/inspect/) affiche la configuration et l’état runtime, [`docker container top`](https://docs.docker.com/reference/cli/docker/container/top/) la vue des processus et [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) les valeurs de ressources en cours. [`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) et [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) vérifient le processus hôte ; le PID hôte et le PID du conteneur peuvent différer.

## Capabilities, Seccomp et Linux Security Modules

Linux décompose les privilèges root classiques en **capabilities**. `CAP_NET_BIND_SERVICE`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` ou `CAP_SYS_PTRACE` ouvrent des opérations très différentes ; `CAP_SYS_ADMIN` confère un pouvoir particulièrement étendu. Un processus qui ne s’exécute pas comme root peut posséder des capabilities, et un processus root peut se les voir presque toutes retirer ([capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)).

Seccomp filtre les appels système. Docker utilise par défaut un profil qui bloque certains syscalls ; `unconfined` supprime cette couche. AppArmor ou SELinux peuvent en outre limiter les accès liés aux objets. Ces contrôles se complètent : une capability autorise une classe d’opérations, Seccomp peut bloquer le syscall correspondant, et le Security Module peut refuser l’accès à un objet précis ([Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/), [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)).

`--privileged` n’est pas une correction pratique d’erreur. Il étend l’accès aux périphériques et aux capabilities et assouplit les profils de sécurité. Si une application ne requiert qu’un port, un périphérique unique ou un chemin en lecture seule, seule cette capacité est accordée et justifiée dans le modèle de menace.

## Root, espaces de noms utilisateur et exploitation rootless

L’UID 0 dans le conteneur est, sans espace de noms utilisateur, le même UID numérique 0 évalué par le noyau hôte. Les espaces de noms limitent les vues et les opérations, mais une erreur de noyau ou de configuration affecte toujours l’hôte. Un espace de noms utilisateur peut mapper les UID du conteneur vers des UID hôte non privilégiés ; les moteurs rootless exécutent le daemon et les conteneurs sans root sur l’hôte ([user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)).

Le mode rootless modifie les prérequis de réseau, de liaison de ports, de cgroups et de stockage ; il s’agit donc d’un modèle d’exploitation, non d’un commutateur universel de durcissement. Indépendamment de cela, les règles suivantes s’appliquent : supprimer les capabilities inutiles, exploiter le système de fichiers racine en lecture seule, monter individuellement les chemins inscriptibles, définir `no-new-privileges` et ne pas intégrer le socket runtime aux workloads. L’accès au socket Docker ou CRI équivaut de fait à l’accès au plan de contrôle de l’hôte.

Après l’isolation des processus et des droits vient l’accessibilité. Un espace de noms réseau propre ne crée d’abord qu’une vue séparée ; le routage, la résolution de noms et les ports publiés doivent être mis en place en plus.

## Réseau : espace de noms, CNI et publication de ports

Selon le mode, un conteneur Linux possède son propre espace de noms réseau. Un moteur le relie souvent à un bridge via une paire veth et effectue du NAT ou une publication de ports. Les réseaux overlay encapsulent le trafic entre hôtes ; les proxies de service ou les chemins de données eBPF répartissent les adresses de service virtuelles. CNI standardise la manière dont un runtime appelle les plugins réseau pour ajouter et retirer un conteneur d’un réseau ; il ne standardise pas l’architecture réseau complète d’un cluster ([CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)).

Quatre adresses sont documentées séparément :

- **adresse d’écoute dans le processus**, par exemple `127.0.0.1:8080` ou `0.0.0.0:8080` dans l’espace de noms ;
- **IP du conteneur/pod**, dont la durée de vie peut être liée à l’instance ;
- **adresse de service ou de load balancer** comme couche d’accès plus stable ;
- **adresse et port hôte publiés**, qui déterminent le pare-feu, le NAT et l’accessibilité externe.

`EXPOSE` dans le Dockerfile ne publie aucun port ; il documente uniquement les ports prévus. La publication de ports est une décision d’exécution. De même, un port hôte ouvert ne prouve pas que la readiness, TLS ou l’authentification de l’application fonctionnent ([Docker – Container networking](https://docs.docker.com/engine/network/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containernetzdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker network inspect $env:CONTAINER_NETWORK
docker container port $env:CONTAINER_NAME
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Test-NetConnection $env:SERVICE_HOST -Port $env:SERVICE_PORT</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker network inspect "$CONTAINER_NETWORK" | jq '.[0].IPAM, .[0].Containers'
docker container port "$CONTAINER_NAME"
ss -lntup
nc -vz "$SERVICE_HOST" "$SERVICE_PORT"</code></pre>
  </div>
</div>

[`docker network inspect`](https://docs.docker.com/reference/cli/docker/network/inspect/) et [`docker container port`](https://docs.docker.com/reference/cli/docker/container/port/) affichent l’affectation du moteur et les ports publiés. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) ou [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affichent les écouteurs ; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient TCP. Pour [DNS](/kb/dns) et [TLS](/kb/tls), leurs propres chemins de diagnostic s’appliquent ensuite.

## Stockage : couche inscriptible, volumes et Bind Mounts

La couche inscriptible du conteneur appartient à l’instance. Elle convient aux modifications temporaires à l’exécution, mais ni aux charges fortement intensives en écriture ni au stockage durable des données. Docker distingue les volumes, Bind Mounts, tmpfs et la couche inscriptible ; les volumes sont gérés par le moteur, les Bind Mounts montent un chemin hôte concret ([Docker – Storage overview](https://docs.docker.com/engine/storage/)).

| Forme | Propriétaire du chemin | Risque typique |
|---|---|---|
| Couche inscriptible | Pilote de stockage/instance du conteneur | Perte lors du remplacement ; coûts de copy-on-write et de capacité |
| Volume | Moteur ou plugin de volume | Nom/plugin/liaison hôte non sauvegardés avec l’image |
| Bind Mount | Administrateur de l’hôte | Chemin hôte, droits, label SELinux et ordre de démarrage |
| tmpfs | Mémoire vive | Perte à l’arrêt ; consommation mémoire et résidus de secrets dans le modèle de swap |
| Service externe | Base de données, stockage objet ou réseau | Réseau, identité, cohérence, quota et plan de reprise propre |

Un montage masque les fichiers existants au chemin cible. Si un montage réseau attendu est absent au démarrage, un chemin de Bind Mount peut se trouver sur l’hôte local et l’application peut y écrire sans s’en apercevoir. Un conteneur démarré avec succès ne prouve donc pas que le bon backend de stockage est actif.

Le standard Container Storage Interface définit une interface orchestrateur/plugin pour les volumes. Il ne garantit ni la mise au repos de l’application, ni la cohérence après crash, ni la capacité de restauration réussie. Les bases de données nécessitent toujours leurs procédures documentées de sauvegarde et de restauration ([Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [Sauvegarde et reprise après sinistre](/kb/backup-dr)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerstorage-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker container inspect $env:CONTAINER_NAME --format '{{json .Mounts}}'
docker volume inspect $env:VOLUME_NAME
docker exec $env:CONTAINER_NAME powershell -NoProfile -Command `
  'Get-Volume | Select-Object DriveLetter,FileSystem,SizeRemaining,Size'
Get-PSDrive -PSProvider FileSystem</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker container inspect "$CONTAINER_NAME" --format '{{json .Mounts}}' | jq .
docker volume inspect "$VOLUME_NAME" | jq .
docker exec "$CONTAINER_NAME" findmnt --json | jq .
docker exec "$CONTAINER_NAME" df -hT</code></pre>
  </div>
</div>

[`docker volume inspect`](https://docs.docker.com/reference/cli/docker/volume/inspect/) fournit le pilote de volume et le point de montage. [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume), [`Get-PSDrive`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive), [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html) et [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) vérifient la situation de stockage effectivement visible et la capacité.

## Configuration et secrets

Les variables d’environnement sont pratiques, mais sont souvent visibles via les sorties Inspect, l’environnement de processus, les rapports de crash ou les bundles de support. Un objet secret dans un orchestrateur n’est pas automatiquement chiffré, rotatif ou caché aux administrateurs de nœuds privilégiés. Kubernetes documente explicitement que les secrets sont stockés par défaut sans chiffrement dans etcd, sauf si Encryption at Rest est configuré ([Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

La configuration est inventoriée en quatre classes :

1. configuration de déploiement non sensible et versionnable ;
2. références de secrets et leur source externe ;
3. état produit à l’exécution, comme les clés hôte, schémas de base de données ou autorités de certification internes ;
4. valeurs par défaut intégrées à l’image, susceptibles de changer lors des mises à jour.

La rotation est une machine à états : fournir le nouveau secret, mettre à jour ou recharger le consommateur, démontrer le bon fonctionnement, révoquer l’ancien secret et tenir compte des caches ou connexions longue durée. Un simple redémarrage du conteneur ne garantit pas une rotation dans le service distant.

## Health, readiness, liveness et startup

L’état du processus, la disponibilité du service et la santé fonctionnelle sont des signaux différents. Docker `HEALTHCHECK` exécute une commande dans le conteneur et enregistre le code de sortie ainsi qu’une sortie limitée. Kubernetes distingue les probes Startup, Readiness et Liveness : Startup protège un démarrage lent contre une Liveness prématurée, Readiness pilote les endpoints, Liveness peut déclencher un redémarrage ([Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/#healthcheck), [Kubernetes – Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

Une bonne probe est peu coûteuse, limitée dans le temps et répond exactement à une question d’exploitation. Une probe Liveness ne doit pas déclencher des redémarrages à chaque incident externe partiel ; sinon elle amplifie les problèmes DNS, de base de données ou de fournisseur. Une probe Readiness doit retirer le service du trafic lorsqu’il ne peut pas accepter de nouveau travail de manière sûre. Les vérifications de bout en bout approfondies relèvent davantage du monitoring que de probes locales exécutées chaque seconde.

Compose `depends_on` contrôle l’ordre de création et de démarrage ; avec des conditions, il peut attendre la santé ou l’exécution réussie d’une tâche de dépendance. Il ne remplace pas la logique de reconnexion : les services peuvent aussi tomber indépendamment après le démarrage ([Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerhealth und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker container inspect $env:CONTAINER_NAME --format '{{json .State.Health}}'
docker container logs --since 30m --timestamps $env:CONTAINER_NAME
docker events --since 30m --filter "container=$env:CONTAINER_NAME"
docker compose ps --all</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker container inspect "$CONTAINER_NAME" --format '{{json .State.Health}}' | jq .
docker container logs --since 30m --timestamps "$CONTAINER_NAME"
docker events --since 30m --filter "container=$CONTAINER_NAME"
journalctl --unit docker --since '-30 minutes' --no-pager</code></pre>
  </div>
</div>

[`docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/) lit le chemin de journalisation configuré, [`docker events`](https://docs.docker.com/reference/cli/docker/system/events/) les événements du moteur et [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) l’état Compose. [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) affiche le contexte du daemon sous systemd. Docker documente séparément les pilotes de journalisation, la rotation et le Dual Logging ; des journaux JSON illimités peuvent remplir l’hôte ([Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)).

Les healthchecks peuvent détecter un processus défaillant et un orchestrateur peut le redémarrer. Les données perdues, une configuration incorrecte ou une dépendance externe défaillante ne sont toutefois pas réparées pour autant.

## Le redémarrage n’est pas une stratégie de reprise

Les politiques de redémarrage réagissent à l’arrêt du processus. Elles ne connaissent ni les données corrompues, ni les files d’attente bloquées, ni les identifiants erronés. Docker indique en outre qu’une Restart Policy ne devient effective que lorsqu’un conteneur a fonctionné avec succès pendant au moins dix secondes ; les arrêts manuels la suppriment jusqu’au redémarrage du daemon ou au démarrage manuel ([Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)).

Les boucles de redémarrage consomment du CPU, génèrent des journaux et peuvent surcharger les services dépendants. Les orchestrateurs utilisent un backoff, mais l’administrateur a néanmoins besoin de la première erreur, du code de sortie, de la raison de l’OOM/de l’éviction, de la dernière configuration et de la chronologie des événements. Un CrashLoop est un état symptomatique, non une cause.

## Orchestration : pod, nœud et état souhaité

Kubernetes regroupe un ou plusieurs conteneurs dans un **pod**. Les conteneurs d’un pod partagent l’espace de noms réseau et peuvent partager des volumes ; ils sont planifiés ensemble. Les deployments gèrent les ReplicaSets et les mises à jour progressives. Sur le nœud, kubelet met en œuvre le PodSpec via CRI, CNI et les plugins de stockage ([Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).

| Niveau | Propriétaire | Incident typique |
|---|---|---|
| Plan de contrôle | API, scheduler, contrôleur | L’état souhaité n’est pas calculé ou planifié |
| Nœud/kubelet | Mise en œuvre locale | Pull d’image, pression disque/PID/mémoire, erreur runtime ou CNI |
| Sandbox de pod | Réseau de pod partagé | Création de sandbox/IP ou perte d’espace de noms |
| Conteneur | Image et processus | Erreur de démarrage, configuration, sortie ou OOM |
| Service/Ingress | Accessibilité et routage | Aucun endpoint prêt, port ou politique incorrecte |
| Volume persistant | Chemin de données | Attach/mount, zone, droits, snapshot ou backend |

La phase `Running` signifie qu’au moins un conteneur principal s’exécute ou démarre ; ce n’est pas un SLA applicatif. Les statuts de conteneur, conditions, événements, probes et état du contrôleur sont évalués ensemble ([Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kubernetes- und CRI-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">kubectl get pod $env:POD -n $env:NAMESPACE -o wide
kubectl describe pod $env:POD -n $env:NAMESPACE
kubectl logs $env:POD -n $env:NAMESPACE --all-containers --previous
crictl ps --all
crictl inspectp $env:POD_SANDBOX_ID</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">kubectl get pod "$POD" -n "$NAMESPACE" -o wide
kubectl describe pod "$POD" -n "$NAMESPACE"
kubectl logs "$POD" -n "$NAMESPACE" --all-containers --previous
crictl ps --all
crictl inspectp "$POD_SANDBOX_ID"</code></pre>
  </div>
</div>

[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/), [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) et [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) affichent les perspectives API, événements et conteneur. [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) examine la vue CRI sur le nœud. Un `kubectl exec` modifie et observe l’instance d’exécution ; il ne remplace ni une image reproductible ni un runbook.

## Compose et systèmes déclaratifs sur hôte unique

Compose décrit les services, réseaux, volumes, secrets, configs et dépendances dans un modèle d’application. Il est précieux pour les systèmes à hôte unique et les flux de développement, mais n’est ni un registre ni un orchestrateur de cluster. La Compose Specification définit le modèle indépendamment d’une CLI donnée ([Compose Specification](https://compose-spec.io/)).

Un dépôt Compose apte à la production contient :

- des digests d’image ou une résolution de tag contrôlée et une plateforme documentée ;
- des montages explicites, un système de fichiers racine en lecture seule et des chemins inscriptibles ;
- la sémantique des ressources, redémarrages, arrêts et healthchecks ;
- la configuration séparée et les références de secrets ;
- les réseaux et ports publiés ;
- les prescriptions de journalisation et de rotation ;
- les procédures de sauvegarde/restauration et de mise à jour en dehors du fichier YAML.

`docker compose config` rend la configuration fusionnée et affiche la résolution des variables. La sortie peut contenir des secrets et est traitée en conséquence ([Docker – `docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)).

## Pull d’image, registre et cache

Un registre distribue du contenu via des manifestes et des blobs. L’authentification, l’autorisation de dépôt, la résolution de tag, le mirror, le cache proxy et les magasins de contenu locaux peuvent échouer séparément. Kubernetes `imagePullPolicy` décide quand kubelet contacte le registre ; même `Always` utilise les couches déjà présentes localement si le digest résolu est disponible ([Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)).

Un déploiement en production journalise le registre, le dépôt, le tag, le digest résolu, la plateforme, la décision concernant la signature/provenance et l’inventaire des nœuds. La garbage collection sur le registre ou le nœud ne doit pas supprimer les digests encore nécessaires à un rollback. L’exploitation air-gapped requiert en plus un processus de miroir, de clés, de révocation et de métadonnées.

## Sécurité de la chaîne d’approvisionnement et de l’exécution

NIST distingue les risques liés aux images, aux registres, aux orchestrateurs, aux conteneurs et aux systèmes d’exploitation hôtes. Le résultat d’un scanner n’est qu’un signal : l’inventaire des paquets peut être incomplet, une CVE peut être inaccessible ou, inversement, une erreur propre à l’application peut rester invisible. Une politique relie provenance, signature, vulnérabilités connues, configuration et contexte d’exécution ([NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final)).

Les Kubernetes Pod Security Standards définissent les profils Privileged, Baseline et Restricted. Restricted exige notamment l’exécution non-root, un profil Seccomp et des capabilities fortement limitées ; les workloads concrets doivent néanmoins être testés fonctionnellement ([Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

Au minimum, il convient de vérifier :

- registre de confiance et digest immuable ;
- provenance de build, signature et source de clés/d’identité contrôlée ;
- image de base minimale et absence de secrets de build dans les couches ;
- non-root, suppression de capabilities, Seccomp/LSM, système de fichiers racine en lecture seule, périphériques et montages limités ;
- aucun socket runtime ou espace de noms hôte sans exception explicite ;
- Network Policies ou pare-feu hôte et egress contrôlé ;
- limites de ressources et de PID contre l’épuisement local ;
- chemin de patch, rebuild, rollout et rollback.

## Conteneurs Windows

Les conteneurs Windows utilisent des images Windows et des mécanismes du noyau Windows. Microsoft distingue **Process Isolation**, où les conteneurs partagent le noyau hôte, et **Hyper-V Isolation**, où chaque conteneur s’exécute dans une VM optimisée avec son propre noyau. Les deux utilisent le même format d’image et les mêmes outils de gestion, mais ont des limites d’isolation et de compatibilité différentes ([Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/), [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)).

Avec Process Isolation, les systèmes d’exploitation hôte et conteneur doivent être compatibles. Hyper-V Isolation peut découpler certaines différences de version, mais augmente les coûts de ressources et de démarrage. Microsoft documente les combinaisons hôte/image prises en charge ; « conteneur Windows » est donc incomplet sans indication de l’image de base, du build et du mode d’isolation ([Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)).

Les nœuds Linux et Windows ne partagent ni le même noyau ni les mêmes binaires d’image. Les clusters multi-OS nécessitent des labels de planification, des DaemonSets adaptés, des plugins réseau/de stockage et des chemins de diagnostic distincts.

## Mises à jour, rollout et rollback

Une mise à jour de conteneur est un **changement d’image assorti d’une transition d’état**. Avant le rollout, il faut vérifier les notes de version, les modifications de schéma, les étapes de migration, les versions minimales des services externes, les nouveaux ports/scopes et la rétrocompatibilité. Un rollback de l’image peut être impossible après une migration de données non rétrocompatible.

Le déroulement contrôlé est le suivant :

1. consigner le digest cible, la signature, la provenance et la décision de scan ;
2. créer une sauvegarde ou un point de reprise testé des données persistantes ;
3. vérifier le diff de configuration et de schéma ;
4. déployer un canary ou des réplicas progressifs avec des probes et SLO réels ;
5. observer la migration des données et la capacité à fonctionner avec des versions mixtes ;
6. vérifier le digest, les instances, les événements, les erreurs, la latence et les ressources ;
7. ne déclencher un rollback que dans les limites de compatibilité des données démontrées.

Ce processus relie [Releases](/kb/releases), [Migration](/kb/migration) et [Backup/DR](/kb/backup-dr). Les outils automatiques de mise à jour de tags sans barrières fonctionnelles ne font que déplacer le moment du changement, du processus de changement vers un bot.

## Sauvegarde et reprise après sinistre

Un service de conteneurs complet comprend davantage que des volumes :

- définitions de déploiement, politiques et objets réseau ;
- digests d’image enregistrés ou miroir de registre restaurable ;
- configuration non sensible et sources de secrets/clés ;
- données persistantes avec une procédure cohérente avec l’application ;
- bases de données externes, files d’attente, stockages objet, dépendances DNS et d’identité ;
- versions de schéma, tâches ouvertes et état de réconciliation ;
- runbooks pour la perte d’un nœud, d’un cluster, d’un registre ou d’un site.

Une archive tar du chemin de volume peut être incohérente lorsqu’une base de données est en cours d’exécution. Les snapshots de stockage nécessitent une sémantique de gel/mise au repos ou propre à la base de données. Un test de restauration reconstruit le service, le réseau et les identités dans un environnement cible propre, démarre avec le digest sauvegardé et vérifie les données fonctionnelles ainsi que les files d’attente ouvertes.

En cas d’incident, la vérification va de l’artefact au processus, puis vers l’extérieur : image, paramètres de démarrage, droits, montages, résolution de noms, chemin réseau et services externes.

## Diagnostic par couches de dépendances

Un incident de conteneur est circonscrit de l’extérieur vers l’intérieur :

1. **État souhaité :** quelle définition et quel digest devraient s’exécuter ?
2. **Placement :** sur quel hôte/nœud, avec quelle plateforme et quelle capacité ?
3. **Image :** pull, authentification, manifeste, plateforme, signature et contenu local ?
4. **Runtime :** sandbox, statut du conteneur, code de sortie, OOM, redémarrage et événements ?
5. **Processus :** PID 1, signaux, utilisateur, capabilities et fichiers ouverts ?
6. **Stockage :** montage attendu, backend, droits, capacité et E/S ?
7. **Réseau :** espace de noms, DNS, route, politique, écouteur, service et TLS ?
8. **Application :** santé, journaux, file d’attente, schéma, identifiant et dépendance externe ?

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Container-Gesamtinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker version
docker info
docker context show
docker compose config --images
docker compose ps --all
docker system df</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker version
docker info
docker context show
docker compose config --images
docker compose ps --all
docker system df
uname -a</code></pre>
  </div>
</div>

[`docker version`](https://docs.docker.com/reference/cli/docker/version/) distingue les versions client et serveur, [`docker info`](https://docs.docker.com/reference/cli/docker/system/info/) affiche le contexte du moteur, du runtime, du stockage et de la sécurité, [`docker context show`](https://docs.docker.com/reference/cli/docker/context/show/) la cible effectivement administrée et [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) la consommation de contenu local. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html) établit, dans l’exemple Unix, le noyau et la plateforme.

## Histoire technique

L’isolation des processus est plus ancienne que les images modernes. `chroot` d’Unix modifiait la racine du système de fichiers d’un processus, mais n’a jamais été conçu comme une limite de sécurité complète. Les jails FreeBSD ont étendu le modèle à la fin des années 1990, respectivement avec FreeBSD 4.0, par des vues hôte et réseau plus isolées ; les Solaris Zones ont associé l’isolation applicative et la gestion des ressources dans le système d’exploitation ([FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/), [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)).

Linux a introduit progressivement les espaces de noms et les Control Groups. LXC a combiné ces mécanismes du noyau en conteneurs système. Docker a popularisé à partir de 2013 les couches d’image, la distribution par registre, les Dockerfiles et une interface développeur cohérente ; il utilisait initialement LXC avant de passer ensuite à sa propre bibliothèque runtime ([Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/), [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

En 2015, des fabricants et fournisseurs de plateformes ont fondé l’Open Container Initiative afin de standardiser ouvertement les formats runtime et image. `runc` est devenu la base de runtime OCI ; `containerd` et CRI-O ont établi les runtimes de haut niveau. Kubernetes a abstrait les runtimes de nœud via CRI, les réseaux via CNI et le stockage via CSI. Aujourd’hui, « conteneur » ne désigne donc pas une pile de produits unique, mais une chaîne de spécifications et d’implémentations interopérables ([Open Container Initiative – Overview](https://opencontainers.org/about/overview/), [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)).

## Checklist administrateur en un coup d’œil

Le conteneur n’est qu’une partie du service. Pour l’autorisation d’exploitation, l’artefact, le runtime, l’hôte, le réseau, le stockage et la reprise sont donc vérifiés comme une chaîne cohérente.

| Question | Preuve d’exploitation |
|---|---|
| Quel artefact s’exécute ? | Registre, dépôt, tag, digest, plateforme, digest de configuration, signature/provenance |
| Quelle chaîne runtime s’applique ? | Moteur/orchestrateur, CRI, runtime de haut niveau, runtime OCI, noyau hôte |
| Quelle isolation est active ? | Espaces de noms, mode d’isolation Windows, mappage UID, capabilities, Seccomp, LSM |
| Où se trouve l’état ? | Couche inscriptible, volume, Bind Mount, tmpfs et services externes par classe de données |
| Qu’est-ce qui est accessible ? | Adresse d’écoute, IP de conteneur/pod, service, port publié, Ingress et politique d’egress |
| Qui possède l’identité ? | UID/SID du processus, compte de service, identifiant de registre, jeton de workload et source de clés |
| Quelles limites s’appliquent ? | CPU, mémoire, PID, E/S, disque, rotation des journaux, quota et éviction de nœud |
| Que signifie sain ? | Processus, Startup, Readiness, Liveness, test fonctionnel et dépendances externes séparés |
| Comment modifier ? | Digest approuvé, diff de schéma/configuration, canary, version mixte, fenêtre de rollback |
| Comment restaurer ? | Définitions, registre/images, secrets/clés, données, files d’attente et test de restauration propre |

## Sources

- [NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)
- [OCI Image Specification](https://specs.opencontainers.org/image-spec/)
- [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)
- [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)
- [OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md)
- [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)
- [Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/)
- [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [SLSA Specification](https://slsa.dev/spec/v1.2/)
- [Sigstore cosign – Signing containers](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)
- [Docker CLI – imagetools inspect](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/)
- [Docker CLI – image inspect](https://docs.docker.com/reference/cli/docker/image/inspect/)
- [Sigstore cosign – Verify](https://docs.sigstore.dev/cosign/verifying/verify/)
- [jq manual](https://jqlang.org/manual/)
- [runc](https://github.com/opencontainers/runc)
- [containerd](https://containerd.io/docs/)
- [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)
- [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Docker CLI – container run / init](https://docs.docker.com/reference/cli/docker/container/run/)
- [Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [Kubernetes – Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Docker CLI – container inspect](https://docs.docker.com/reference/cli/docker/container/inspect/)
- [Docker CLI – container top](https://docs.docker.com/reference/cli/docker/container/top/)
- [Docker CLI – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Microsoft Learn – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [ps(1)](https://man7.org/linux/man-pages/man1/ps.1.html)
- [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/)
- [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)
- [user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)
- [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [Docker – Container networking](https://docs.docker.com/engine/network/)
- [Docker CLI – network inspect](https://docs.docker.com/reference/cli/docker/network/inspect/)
- [Docker CLI – container port](https://docs.docker.com/reference/cli/docker/container/port/)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Docker – Storage overview](https://docs.docker.com/engine/storage/)
- [Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md)
- [Docker CLI – volume inspect](https://docs.docker.com/reference/cli/docker/volume/inspect/)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume)
- [Microsoft Learn – Get-PSDrive](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive)
- [findmnt(8)](https://man7.org/linux/man-pages/man8/findmnt.8.html)
- [GNU coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/)
- [Kubernetes – Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)
- [Docker CLI – container logs](https://docs.docker.com/reference/cli/docker/container/logs/)
- [Docker CLI – events](https://docs.docker.com/reference/cli/docker/system/events/)
- [Docker CLI – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)
- [Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)
- [Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [Kubernetes – crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Compose Specification](https://compose-spec.io/)
- [Docker CLI – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/)
- [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)
- [Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)
- [Docker CLI – version](https://docs.docker.com/reference/cli/docker/version/)
- [Docker CLI – info](https://docs.docker.com/reference/cli/docker/system/info/)
- [Docker CLI – context show](https://docs.docker.com/reference/cli/docker/context/show/)
- [Docker CLI – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [GNU coreutils – uname](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html)
- [FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/)
- [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)
- [Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/)
- [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Open Container Initiative – Overview](https://opencontainers.org/about/overview/)
- [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
