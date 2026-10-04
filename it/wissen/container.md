---
title: "Container: immagini, runtime e funzionamento affidabile"
blatt: "container"
description: "Container per amministratori di infrastruttura e messaggistica: artefatti OCI, isolamento di runtime e kernel, namespace e cgroup, rete e storage, orchestrazione, identità e supply chain, risorse, health, logging, recovery, container Windows e storia tecnica."
fakten:
  - label: Ruolo di sistema
    wert: insieme di processi isolati con filesystem root pacchettizzato e limiti espliciti per risorse e I/O
    href: https://csrc.nist.gov/pubs/sp/800/190/final
  - label: Standard dell’artefatto
    wert: OCI Image Specification · manifest · configurazione · layer · descrittori
    href: https://specs.opencontainers.org/image-spec/
  - label: Standard di runtime
    wert: OCI Runtime Specification · bundle · config.json · ciclo di vita
    href: https://specs.opencontainers.org/runtime-spec/
  - label: Base Linux
    wert: namespace · cgroup · capabilities · LSM · Seccomp
    href: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - label: Base Windows
    wert: isolamento Process o Hyper-V con immagini di container Windows
    href: https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/
  - label: Persistenza
    wert: volume, bind mount o servizio esterno; il writable layer non è una strategia di recovery
    href: https://docs.docker.com/engine/storage/
  - label: Rete
    wert: namespace di rete/vNIC · bridge/overlay · pubblicazione delle porte · DNS/service discovery
    href: https://github.com/containernetworking/cni/blob/main/SPEC.md
  - label: Risorse
    wert: CPU · memoria · PID · I/O; considerare separatamente limiti e prenotazioni
    href: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - label: Orchestrazione
    wert: stato desiderato, scheduling, restart, rollout e service discovery; non fanno parte dell’immagine
    href: https://kubernetes.io/docs/concepts/workloads/pods/
  - label: Supply chain
    wert: digest · registry · provenance/attestation · firma · policy
    href: https://slsa.dev/spec/v1.2/
  - label: Stato operativo
    wert: processo · health · risorse · rete · storage · log · eventi · stato desiderato
    href: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
  - label: Oggetto di recovery
    wert: definizione del deployment, immagini/digest, configurazione, secret/chiavi e dati persistenti
    href: https://csrc.nist.gov/pubs/sp/800/190/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - paperless-ngx
  - home-assistant
translationSourceHash: 7091efc5400536d4b1b40e368e564d2ab0d5332766f61e0fdaf976e723bff142
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:28:42.097Z
translationReview: required
---

# Container: immagini, runtime e funzionamento affidabile

Un container non è un piccolo server e un'immagine non è un backup. Dal punto di vista tecnico, un runtime per container esegue normali processi con un filesystem predisposto, viste isolate delle risorse del kernel e limiti di risorse e sicurezza impostati. Il kernel host rimane parte di ogni operazione. Questa architettura rende gli ambienti applicativi riproducibili e rapidamente sostituibili; tuttavia, sposta la responsabilità su origine delle immagini, runtime, rete, persistenza, secret, controllo delle risorse e orchestrazione.

Per gli amministratori, la distinzione più importante è quella tra **artefatto**, **istanza di runtime** e **stato operativo**. Un'immagine descrive il filesystem avviabile e i metadati. Un container è un'esecuzione concreta. Database, code, chiavi, configurazione e dati di audit hanno cicli di vita propri. Chi tratta questi livelli congiuntamente come «il container Docker» non può né circoscrivere un problema né pianificare completamente un ripristino.

La spiegazione parte dall'immagine come artefatto distribuibile e ne segue il percorso attraverso Engine e runtime fino a kernel, rete e storage. Orchestrazione, sicurezza, diagnosi e recovery si basano su questo flusso.

## Inquadramento tecnico

NIST descrive i container applicativi come una forma di virtualizzazione del sistema operativo: le applicazioni condividono il kernel, mentre l'isolamento e il controllo delle risorse si basano su meccanismi del sistema operativo. Le macchine virtuali virtualizzano invece l'hardware per un proprio kernel guest. Container e VM possono essere combinati; la scelta modifica superficie di attacco, compatibilità, costo di avvio e ambito di guasto ([NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)).

Un tipico percorso Linux è composto da:

1. registry o content store locale con manifest OCI, configurazioni e layer;
2. engine o orchestratore che determina stato desiderato, rete, mount e policy;
3. runtime di alto livello come `containerd` o CRI-O per la gestione di immagini e container;
4. runtime OCI come `runc`, che crea il processo a partire da bundle e `config.json`;
5. kernel host con namespace, cgroup, capabilities, Seccomp e un Linux Security Module;
6. processi dei container che interagiscono con il loro ambiente attraverso filesystem virtuali o montati, socket e dispositivi.

La Open Container Initiative standardizza separatamente le componenti di immagini, runtime e distribuzione. Di conseguenza, un'immagine OCI può essere elaborata da diversi engine e runtime, senza che rete, orchestrazione o backup dei dati siano standardizzati automaticamente ([OCI Image Specification](https://specs.opencontainers.org/image-spec/), [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/), [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-container.svg?v=20260813" title="Interaktive Infografik: Container von OCI-Artefakt und Registry über Engine, Runtime und Kernelisolation bis Netzwerk, Storage, Identität, Supply Chain und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-container.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Immagine: manifest, configurazione e layer

Un'immagine OCI è un grafo diretto di contenuti. Un **manifest** fa riferimento, tramite descrittori e digest crittografici, a una configurazione dell'immagine e a layer di filesystem ordinati. Un Image Index facoltativo può raggruppare più manifest per diversi sistemi operativi e architetture. Media type, digest e dimensione fanno parte del descrittore; i tag appartengono invece alla risoluzione dei nomi del registry e non sono un'identità immutabile ([OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md), [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)).

I layer contengono modifiche al filesystem. All'avvio vengono assemblati tramite uno storage driver in una vista root comune e integrati da un layer del container scrivibile. Un secret o il contenuto di un pacchetto eliminato può rimanere presente in un layer precedente. Le build multi-stage riducono strumenti di build e artefatti intermedi nell'immagine finale, ma non sostituiscono la verifica di secret e provenienza ([Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/), [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)).

### Tag, digest e selezione della piattaforma

Un tag come `stable`, `3` o `latest` può essere spostato a un altro digest del manifest. Una produzione riproducibile fissa pertanto il digest approvato oppure documenta almeno il digest risolto durante il rollout. Per immagini multi-arch, il client seleziona una piattaforma dall'indice; architettura errata, funzionalità CPU mancanti o emulazione possono portare a un comportamento differente nonostante lo stesso tag.

Un digest risponde a «quali byte?», non a «chi li ha creati?» o «sono sicuri?». Firme e attestazioni possono vincolare identità, provenienza della build e materiali. SLSA descrive la provenance come informazione verificabile su dove, quando e come è stato prodotto un artefatto; `cosign` di Sigstore può firmare e verificare artefatti di container ([SLSA Specification](https://slsa.dev/spec/v1.2/), [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Imageinventar">
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

[`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/) mostra liste di manifest e piattaforme, [`docker image inspect`](https://docs.docker.com/reference/cli/docker/image/inspect/) mostra i metadati locali dell'immagine. [`cosign verify`](https://docs.sigstore.dev/cosign/verifying/verify/) verifica firma e identità prevista; [`jq`](https://jqlang.org/manual/) formatta JSON nell'esempio Unix.

Dal'immagine immutabile, il runtime genera il container effettivamente in esecuzione. Solo allora si combinano writable layer, namespace, cgroup, mount e parametri di processo.

## Runtime, bundle e ciclo di vita del container

La OCI Runtime Specification descrive un container come ambiente per un processo. Un **bundle** contiene `config.json` e un filesystem root. La configurazione definisce argomenti del processo, ambiente, utente, mount, namespace, risorse e ulteriori opzioni della piattaforma. Il ciclo di vita distingue creazione, avvio, terminazione ed eliminazione; un processo con stato `created` non è ancora in esecuzione ([OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)).

`runc` è un'implementazione di riferimento di questa interfaccia di basso livello. `containerd` gestisce tramite essa immagini, snapshot, container, task ed eventi. Docker Engine fornisce un'API di livello superiore con rete, volumi, build e modello operativo. Su un nodo Kubernetes comunica tramite la **Container Runtime Interface (CRI)** con un runtime compatibile; kubelet non usa semplicemente la CLI Docker ([runc](https://github.com/opencontainers/runc), [containerd](https://containerd.io/docs/), [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)).

Questi livelli hanno stati separati. Un Pod Kubernetes può riportare `Running` mentre un processo di container è in un ciclo di restart; un engine può conoscere un container il cui task runtime non è più attivo; un processo può essere in esecuzione mentre il servizio non accetta traffico. La diagnosi inizia quindi dalla domanda: **stato desiderato, oggetto engine, task runtime, processo o applicazione?**

## Namespace: vista isolata, non una macchina separata

I namespace Linux isolano risorse globali in viste separate. Le manpage del kernel elencano tra gli altri namespace Mount, PID, Network, IPC, UTS, User, cgroup e Time. Un processo può appartenere esattamente a un namespace per ciascun tipo di namespace; le relazioni possono essere analizzate tramite `/proc/<pid>/ns` ([namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)).

| Namespace | Vista isolata | Rilevanza per l'amministrazione |
|---|---|---|
| Mount | Punti di mount e filesystem root | Bind mount, propagation, percorsi host mascherati |
| PID | ID e gerarchia dei processi | PID 1, comportamento di segnali e reaping |
| Network | Interfacce, route, porte, stato del firewall | Il container può essere in ascolto internamente senza essere raggiungibile dall'esterno |
| UTS | Nome host e nome di dominio | Identità cosmetica, non DNS né security principal |
| IPC | IPC System V e code di messaggi POSIX | IPC condiviso solo con configurazione esplicita |
| User | Mappatura di UID/GID | Il root del container può essere mappato esternamente come non privilegiato |
| cgroup | Vista della gerarchia cgroup | Da solo non impedisce l'uso delle risorse |
| Time | Offset selezionati del tempo monotono/di boot | Raramente usato; non è un isolamento generale del fuso orario |

L'isolamento è configurabile. Rete host, PID host, `--privileged`, condivisioni di dispositivi, bind mount estesi o capabilities aggiuntive aprono deliberatamente i confini. Il nome del container e un UID interno non sono un security principal tra host, registry e orchestratore.

### PID 1, segnali e terminazione pulita

Nel namespace PID, il primo processo assume compiti speciali. Riceve i segnali in modo diverso e deve raccogliere i processi figli orfani. I wrapper shell che non avviano il programma effettivo con `exec` possono inghiottire i segnali di terminazione. Gli orchestratori inviano di solito prima un segnale di terminazione, attendono un periodo di grazia e poi forzano la fine. L'applicazione deve interrompere il nuovo lavoro, completare in modo limitato le operazioni in corso e scaricare gli stati.

Docker può usare un piccolo processo init con `--init`; ciò non ripara l'applicazione, ma rende più espliciti l'inoltro dei segnali e il reaping dei figli ([Docker run reference – `--init`](https://docs.docker.com/reference/cli/docker/container/run/#init)). Per code di posta, database e indicizzatori, il tempo massimo di shutdown sicuro deve far parte del modello di deployment.

## cgroup: controllo e contabilizzazione

I cgroup raggruppano processi e applicano controller alle risorse. In cgroup v2, i processi formano una gerarchia unificata. I controller CPU, memoria, I/O e PID hanno semantiche diverse: un limite CPU limita la velocità, un limite di memoria può causare un OOM kill, un limite PID impedisce nuovi processi o thread e i limiti I/O dipendono dal percorso effettivo del dispositivo a blocchi ([Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)).

**Request/Reservation** e **Limit** non devono essere confusi. Kubernetes usa le request per lo scheduling e i limit per i limiti di runtime; per la memoria, il limit è reattivo e viene applicato dal kernel sotto pressione. Un Pod può essere evicted o terminato nonostante sufficiente capacità complessiva, a causa della pressione del nodo o del cgroup ([Kubernetes – Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerprozess- und Ressourceninventar">
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

[`docker container inspect`](https://docs.docker.com/reference/cli/docker/container/inspect/) mostra configurazione e stato runtime, [`docker container top`](https://docs.docker.com/reference/cli/docker/container/top/) la vista dei processi e [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) i valori delle risorse in esecuzione. [`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) e [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) verificano il processo host; PID host e PID del container possono essere diversi.

## Capabilities, Seccomp e Linux Security Modules

Linux scompone i classici privilegi root in **capabilities**. `CAP_NET_BIND_SERVICE`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` o `CAP_SYS_PTRACE` abilitano operazioni molto diverse; `CAP_SYS_ADMIN` comprende poteri particolarmente ampi. Un processo che non viene eseguito come root può avere capabilities, e a un processo root possono essere rimosse quasi tutte ([capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)).

Seccomp filtra le chiamate di sistema. Docker utilizza per impostazione predefinita un profilo che blocca syscall selezionate; `unconfined` rimuove questo livello. AppArmor o SELinux possono inoltre limitare gli accessi relativi agli oggetti. Questi controlli si completano: una capability consente una classe di operazioni, Seccomp può bloccare la syscall corrispondente e il Security Module può negare l'accesso a un oggetto concreto ([Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/), [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)).

`--privileged` non è una comoda soluzione ai problemi. Estende l'accesso a dispositivi e capabilities e allenta i profili di sicurezza. Se un'applicazione necessita solo di una porta, di un singolo dispositivo o di un percorso in sola lettura, viene abilitata esattamente tale capacità e giustificata nel threat model.

## Root, namespace utente ed esecuzione rootless

UID 0 nel container, senza user namespace, è lo stesso UID numerico 0 valutato dal kernel host. I namespace limitano visibilità e operazioni, ma un errore del kernel o di configurazione colpisce comunque l'host. Un user namespace può mappare gli UID del container su UID host non privilegiati; gli engine rootless eseguono daemon e container senza root sull'host ([user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)).

Rootless modifica i prerequisiti per rete, binding delle porte, cgroup e storage e costituisce quindi un modello operativo, non un interruttore universale di hardening. Indipendentemente da ciò, valgono le seguenti regole: rimuovere le capabilities non necessarie, eseguire il filesystem root in sola lettura, montare separatamente i percorsi scrivibili, impostare `no-new-privileges` e non collegare il socket runtime ai workload. L'accesso al socket Docker o CRI equivale di fatto all'accesso al control plane dell'host.

Dopo l'isolamento di processo e privilegi segue la raggiungibilità. Un namespace di rete separato crea inizialmente solo una vista separata; routing, risoluzione dei nomi e porte pubblicate devono essere configurati in aggiunta.

## Rete: namespace, CNI e pubblicazione delle porte

A seconda della modalità, un container Linux dispone di un proprio namespace di rete. Un engine lo collega spesso tramite una coppia veth a un bridge ed esegue NAT o pubblicazione delle porte. Le reti overlay incapsulano il traffico tra host; service proxy o datapath eBPF distribuiscono indirizzi di servizio virtuali. CNI standardizza il modo in cui un runtime richiama plugin di rete per aggiungere e rimuovere un container da una rete; non standardizza l'intera architettura di rete di un cluster ([CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)).

Quattro indirizzi vengono documentati separatamente:

- **indirizzo di ascolto nel processo**, ad esempio `127.0.0.1:8080` o `0.0.0.0:8080` nel namespace;
- **IP del container/Pod**, la cui durata può essere legata all'istanza;
- **indirizzo del servizio o del load balancer** come livello di accesso più stabile;
- **indirizzo e porta host pubblicati**, che determinano firewall, NAT e raggiungibilità esterna.

`EXPOSE` nel Dockerfile non pubblica una porta; documenta soltanto le porte previste. La pubblicazione delle porte è una decisione di runtime. Analogamente, una porta host aperta non prova che readiness, TLS o autenticazione dell'applicazione funzionino ([Docker – Container networking](https://docs.docker.com/engine/network/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containernetzdiagnose">
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

[`docker network inspect`](https://docs.docker.com/reference/cli/docker/network/inspect/) e [`docker container port`](https://docs.docker.com/reference/cli/docker/container/port/) mostrano l'associazione dell'engine e le porte pubblicate. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano rispettivamente i listener; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano TCP. Per [DNS](/kb/dns) e [TLS](/kb/tls) valgono quindi i rispettivi percorsi diagnostici.

## Storage: writable layer, volumi e bind mount

Il layer scrivibile del container appartiene all'istanza. È adatto a modifiche temporanee in runtime, ma non a carichi di scrittura elevati né come archiviazione dati permanente. Docker distingue tra volumi, bind mount, tmpfs e writable layer; i volumi sono gestiti dall'engine, i bind mount collegano uno specifico percorso host ([Docker – Storage overview](https://docs.docker.com/engine/storage/)).

| Forma | Proprietario del percorso | Rischio tipico |
|---|---|---|
| Writable layer | Storage driver/istanza del container | Perdita alla sostituzione; costi di copy-on-write e capacità |
| Volume | Engine o plugin di volume | Nome/plugin/legame con host non viene salvato con l'immagine |
| Bind mount | Amministratore host | Percorso host, diritti, etichetta SELinux e ordine di avvio |
| tmpfs | Memoria di lavoro | Perdita allo stop; consumo di memoria e residui di secret nel modello di swap |
| Servizio esterno | Database, object storage o storage di rete | Rete, identità, coerenza, quota e piano di recovery proprio |

Un mount copre i file esistenti nel percorso di destinazione. Se un mount di rete previsto manca all'avvio, un percorso bind mount può risiedere sull'host locale e l'applicazione può scrivervi senza accorgersene. Un container avviato correttamente non dimostra quindi che sia attivo il backend di storage corretto.

Lo standard Container Storage Interface definisce un'interfaccia orchestratore/plugin per i volumi. Non garantisce quiescenza dell'applicazione, coerenza dopo un crash o capacità di ripristino riuscita. I database richiedono comunque le procedure documentate di backup e ripristino ([Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [Backup e Disaster Recovery](/kb/backup-dr)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerstorage-Inventar">
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

[`docker volume inspect`](https://docs.docker.com/reference/cli/docker/volume/inspect/) fornisce driver del volume e mountpoint. [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume), [`Get-PSDrive`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive), [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html) e [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) verificano la situazione e la capacità di storage effettivamente visibili.

## Configurazione e secret

Le variabili d'ambiente sono comode, ma sono spesso visibili tramite output Inspect, ambiente del processo, crash report o bundle di supporto. Un oggetto secret in un orchestratore non è inoltre automaticamente cifrato, ruotato o nascosto agli amministratori privilegiati del nodo. Kubernetes documenta esplicitamente che i secret vengono archiviati in etcd non cifrati per impostazione predefinita, salvo configurazione della cifratura at rest ([Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

La configurazione viene inventariata in quattro classi:

1. configurazione di deployment non sensibile e versionabile;
2. riferimenti ai secret e relativa fonte esterna;
3. stato generato in fase di runtime, come host key, schemi di database o CA interne;
4. impostazioni predefinite integrate nell'immagine, che possono cambiare con gli aggiornamenti.

La rotazione è una macchina a stati: fornire il nuovo secret, aggiornare o ricaricare il consumer, dimostrarne il funzionamento, revocare il vecchio secret e considerare cache e connessioni di lunga durata. Il solo restart del container non garantisce la rotazione nel servizio remoto.

## Health, readiness, liveness e startup

Stato del processo, disponibilità del servizio e salute funzionale sono segnali diversi. Docker `HEALTHCHECK` esegue un comando nel container e memorizza exit code e output limitato. Kubernetes distingue startup, readiness e liveness probe: startup protegge un avvio lento da liveness premature, readiness controlla gli endpoint, liveness può attivare un restart ([Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/#healthcheck), [Kubernetes – Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

Una buona probe è economica, limitata nel tempo e risponde esattamente a una domanda operativa. Una liveness probe non deve attivare restart a ogni problema parziale esterno; altrimenti amplifica problemi DNS, database o provider. Una readiness probe può rimuovere il servizio dal traffico se non può accettare nuovo lavoro in sicurezza. I controlli end-to-end approfonditi appartengono più al monitoraggio che a probe locali eseguite ogni secondo.

Compose `depends_on` controlla l'ordine di creazione e avvio; con condizioni può attendere l'health o la conclusione riuscita di un job dipendente. Non sostituisce la logica di riconnessione: i servizi possono guastarsi indipendentemente anche dopo l'avvio ([Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Containerhealth und Logs">
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

[`docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/) legge il percorso di logging configurato, [`docker events`](https://docs.docker.com/reference/cli/docker/system/events/) gli eventi dell'engine e [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) lo stato Compose. [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) mostra il contesto del daemon con systemd. Docker documenta separatamente logging driver, rotazione e dual logging; i log JSON senza limiti possono riempire l'host ([Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)).

Gli health check possono riconoscere un processo difettoso e un orchestratore può riavviarlo. Tuttavia, dati persi, configurazione errata o una dipendenza esterna guasta non vengono riparati in questo modo.

## Il restart non è una strategia di recovery

Le restart policy reagiscono alla fine del processo. Non conoscono né dati danneggiati né code bloccate né credenziali errate. Docker segnala inoltre che una restart policy diventa efficace solo se un container è stato eseguito correttamente per almeno dieci secondi; gli stop manuali la sopprimono fino al restart del daemon o all'avvio manuale ([Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)).

I cicli di restart consumano CPU, producono log e possono sovraccaricare i servizi dipendenti. Gli orchestratori utilizzano un backoff, ma l'amministratore necessita comunque del primo errore, dell'exit code, del motivo OOM/eviction, dell'ultima configurazione e della cronologia degli eventi. Un CrashLoop è uno stato sintomatico, non una causa.

## Orchestrazione: Pod, nodo e stato desiderato

Kubernetes raggruppa uno o più container in un **Pod**. I container di un Pod condividono il namespace di rete e possono condividere volumi; vengono pianificati insieme. I deployment gestiscono ReplicaSet e aggiornamenti graduali. Sul nodo, kubelet implementa il PodSpec tramite CRI, CNI e plugin di storage ([Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).

| Livello | Proprietario | Problema tipico |
|---|---|---|
| Control plane | API, scheduler, controller | Lo stato desiderato non viene calcolato o pianificato |
| Nodo/kubelet | Implementazione locale | Image pull, pressione Disk/PID/Memory, errore runtime o CNI |
| Pod sandbox | Rete Pod condivisa | Creazione di sandbox/IP o perdita del namespace |
| Container | Immagine e processo | Errore di avvio, configurazione, exit o OOM |
| Service/Ingress | Raggiungibilità e routing | Nessun endpoint ready, porta o policy errata |
| Persistent Volume | Percorso dati | Attach/mount, zona, diritti, snapshot o backend |

La fase `Running` indica che almeno un container primario è in esecuzione o in avvio; non è un SLA applicativo. Stato dei container, condizioni, eventi, probe e stato dei controller vengono valutati insieme ([Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kubernetes- und CRI-Diagnose">
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

[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/), [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) e [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) mostrano la prospettiva di API, eventi e container. [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) analizza la vista CRI sul nodo. Un `kubectl exec` modifica e osserva l'istanza runtime; non sostituisce un'immagine riproducibile o un runbook.

## Compose e sistemi dichiarativi su singolo host

Compose descrive servizi, reti, volumi, secret, config e dipendenze in un modello applicativo. È utile per host singoli e flussi di sviluppo, ma non è né un registry né un orchestratore di cluster. La Compose Specification definisce il modello indipendentemente da una CLI specifica ([Compose Specification](https://compose-spec.io/)).

Un repository Compose pronto per la produzione contiene:

- digest delle immagini o risoluzione controllata dei tag e piattaforma documentata;
- mount espliciti, filesystem root in sola lettura e percorsi scrivibili;
- semantica di risorse, restart, stop e health;
- configurazione separata e riferimenti ai secret;
- reti e porte pubblicate;
- specifiche di logging e rotazione;
- procedure di backup/ripristino e aggiornamento esterne al file YAML.

`docker compose config` esegue il rendering della configurazione unita e mostra la risoluzione delle variabili. L'output può contenere secret e viene trattato di conseguenza ([Docker – `docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)).

## Image pull, registry e cache

Un registry distribuisce contenuti attraverso manifest e blob. Autenticazione, autorizzazione del repository, risoluzione dei tag, mirror, cache proxy e content store locali possono fallire separatamente. Kubernetes `imagePullPolicy` decide quando kubelet contatta il registry; anche `Always` usa layer disponibili localmente se il digest risolto è già presente ([Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)).

Un rollout produttivo registra registry, repository, tag, digest risolto, piattaforma, decisione di firma/provenance e inventario del nodo. La garbage collection nel registry o sul nodo non deve eliminare digest ancora necessari per il rollback. L'esercizio air-gapped richiede inoltre un processo per mirror, chiavi, revoca e metadati.

## Sicurezza della supply chain e del runtime

NIST distingue i rischi in immagini, registry, orchestratori, container e sistemi operativi host. Il risultato di uno scanner è solo un segnale: l'inventario dei pacchetti può essere incompleto, una CVE può non essere raggiungibile o, viceversa, un errore specifico dell'applicazione può rimanere invisibile. Una policy collega provenienza, firma, vulnerabilità note, configurazione e contesto runtime ([NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final)).

I Pod Security Standards di Kubernetes definiscono i profili Privileged, Baseline e Restricted. Restricted richiede tra l'altro l'esecuzione non-root, un profilo Seccomp e capabilities fortemente limitate; i workload concreti devono comunque essere testati funzionalmente ([Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

Vengono verificati almeno:

- registry affidabile e digest immutabile;
- provenance della build, firma e fonte controllata di chiavi/identità;
- base image minima e nessun secret di build nei layer;
- non-root, rimozione delle capabilities, Seccomp/LSM, filesystem root in sola lettura, dispositivi e mount limitati;
- nessun socket runtime o namespace host senza eccezione esplicita;
- Network Policy oppure firewall host ed egress controllato;
- limiti di risorse e PID contro l'esaurimento locale;
- percorso di patch, rebuild, rollout e rollback.

## Container Windows

I container Windows utilizzano immagini Windows e meccanismi del kernel Windows. Microsoft distingue **Process Isolation**, in cui i container condividono il kernel host, e **Hyper-V Isolation**, in cui ciascun container viene eseguito in una VM ottimizzata con kernel proprio. Entrambi usano lo stesso formato di immagine e gli stessi strumenti di gestione, ma confini di isolamento e compatibilità diversi ([Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/), [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)).

Con Process Isolation, sistema operativo host e container devono essere compatibili. Hyper-V Isolation può separare determinate differenze di versione, ma aumenta i costi di risorse e avvio. Microsoft documenta le combinazioni supportate di host e immagini; «Windows Container» è pertanto incompleto senza indicazione di base image, build e isolamento ([Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)).

I nodi Linux e Windows non condividono lo stesso kernel né gli stessi binari delle immagini. I cluster multi-OS richiedono label di scheduling, DaemonSet appropriati, plugin di rete/storage e percorsi diagnostici diversi.

## Aggiornamenti, rollout e rollback

Un aggiornamento del container è un **cambio di immagine più una transizione di stato**. Prima del rollout vengono verificati release note, modifiche dello schema, passaggi di migrazione, versioni minime dei servizi esterni, nuove porte/scope e compatibilità all'indietro. Il rollback dell'immagine può essere impossibile dopo una migrazione dati non retrocompatibile.

Il flusso controllato è il seguente:

1. registrare digest di destinazione, firma, provenance e decisione dello scan;
2. creare un backup o un punto di recovery testato dei dati persistenti;
3. verificare il diff di configurazione e schema;
4. distribuire canary o repliche graduali con probe e SLO reali;
5. osservare la migrazione dei dati e la capacità di versioni miste;
6. verificare digest, istanze, eventi, errori, latenza e risorse;
7. attivare il rollback solo entro la compatibilità dati comprovata.

Questo processo unisce [Release](/kb/releases), [Migrazione](/kb/migration) e [Backup/DR](/kb/backup-dr). Gli updater automatici dei tag senza gate funzionali spostano soltanto il momento della modifica dal processo di change a un bot.

## Backup e Disaster Recovery

Un servizio container completo è composto da più elementi dei soli volumi:

- definizioni di deployment, policy e oggetti di rete;
- digest delle immagini registrati o un mirror di registry ripristinabile;
- configurazione non sensibile e fonti di secret/chiavi;
- dati persistenti con procedura coerente con l'applicazione;
- database esterni, code, object store, dipendenze DNS e di identità;
- versioni di schema, job aperti e stato di riconciliazione;
- runbook per perdita di nodo, cluster, registry e sede.

Un archivio tar del percorso del volume può essere incoerente con un database in esecuzione. Gli snapshot di storage richiedono semantica di freeze/quiesce o specifica del database. Un test di ripristino ricostruisce servizio, rete e identità in un ambiente di destinazione pulito, avvia con il digest salvato e verifica dati funzionali e code aperte.

In caso di problemi si verifica dall'artefatto al processo e poi verso l'esterno: immagine, parametri di avvio, diritti, mount, risoluzione dei nomi, percorso di rete e servizi esterni.

## Diagnosi per livelli di dipendenza

Un incidente del container viene circoscritto dall'esterno verso l'interno:

1. **Stato desiderato:** quale definizione e quale digest devono essere in esecuzione?
2. **Posizionamento:** su quale host/nodo, con quale piattaforma e capacità?
3. **Immagine:** pull, autenticazione, manifest, piattaforma, firma e contenuto locale?
4. **Runtime:** sandbox, stato del container, exit code, OOM, restart ed eventi?
5. **Processo:** PID 1, segnali, utente, capabilities e file aperti?
6. **Storage:** mount previsto, backend, diritti, capacità e I/O?
7. **Rete:** namespace, DNS, route, policy, listener, servizio e TLS?
8. **Applicazione:** health, log, coda, schema, credenziale e dipendenza esterna?

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Container-Gesamtinventar">
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

[`docker version`](https://docs.docker.com/reference/cli/docker/version/) distingue versione client e server, [`docker info`](https://docs.docker.com/reference/cli/docker/system/info/) mostra il contesto di engine, runtime, storage e sicurezza, [`docker context show`](https://docs.docker.com/reference/cli/docker/context/show/) la destinazione effettivamente amministrata e [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) il consumo di contenuti locali. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html) documenta kernel e piattaforma nell'esempio Unix.

## Storia tecnica

L'isolamento dei processi è precedente alle immagini moderne. Il `chroot` Unix modificava la radice del filesystem di un processo, ma non è mai stato concepito come confine di sicurezza completo. Le FreeBSD Jails hanno esteso il modello alla fine degli anni 1990, rispettivamente con FreeBSD 4.0, con viste host e di rete più isolate; le Solaris Zones hanno combinato isolamento delle applicazioni e gestione delle risorse nel sistema operativo ([FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/), [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)).

Linux ha introdotto gradualmente namespace e Control Groups. LXC ha combinato questi meccanismi del kernel in container di sistema. Docker ha reso popolari dal 2013 image layer, distribuzione tramite registry, Dockerfile e un'interfaccia coerente per gli sviluppatori; inizialmente utilizzava LXC e in seguito è passato a una propria libreria runtime ([Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/), [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

Nel 2015 produttori e fornitori di piattaforme hanno fondato la Open Container Initiative per standardizzare in modo aperto formati di runtime e immagini. `runc` è diventata la base del runtime OCI; `containerd` e CRI-O hanno consolidato runtime di alto livello. Kubernetes ha astratto i runtime dei nodi tramite CRI, le reti tramite CNI e lo storage tramite CSI. Oggi «container» non indica quindi un singolo stack di prodotti, bensì una catena di specifiche e implementazioni interoperabili ([Open Container Initiative – Overview](https://opencontainers.org/about/overview/), [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)).

## Checklist dell'amministratore a colpo d'occhio

Il container è solo una parte del servizio. Per l'approvazione operativa vengono quindi verificati come catena coerente artefatto, runtime, host, rete, storage e recovery.

| Domanda | Evidenza operativa |
|---|---|
| Quale artefatto è in esecuzione? | Registry, repository, tag, digest, piattaforma, digest di configurazione, firma/provenance |
| Quale catena runtime si applica? | Engine/orchestratore, CRI, runtime di alto livello, runtime OCI, kernel host |
| Quale isolamento è attivo? | Namespace, modalità di isolamento Windows, mappatura UID, capabilities, Seccomp, LSM |
| Dove risiede lo stato? | Writable layer, volume, bind mount, tmpfs e servizi esterni per ciascuna classe di dati |
| Cosa è raggiungibile? | Indirizzo di ascolto, IP container/Pod, servizio, porta pubblicata, ingress e policy di egress |
| Chi possiede l'identità? | UID/SID del processo, service account, credenziale registry, token del workload e fonte delle chiavi |
| Quali limiti si applicano? | CPU, memoria, PID, I/O, disco, rotazione log, quota ed eviction del nodo |
| Che cosa significa sano? | Processo, startup, readiness, liveness, test funzionale e dipendenze esterne separati |
| Come viene modificato? | Digest approvato, diff di schema/configurazione, canary, versione mista, finestra di rollback |
| Come viene ripristinato? | Definizioni, registry/immagini, secret/chiavi, dati e test di ripristino pulito |

## Fonti

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
