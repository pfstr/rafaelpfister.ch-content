---
title: "Containere: bilder, kjøretid og robust drift"
blatt: "container"
description: "Containere for infrastruktur- og meldingsadministratorer: OCI-artefakter, runtime- og kjerneisolasjon, namespaces og cgroups, nettverk og lagring, orkestrering, identitet og forsyningskjede, ressurser, helse, logging, gjenoppretting, Windows-containere og teknisk historie."
fakten:
  - label: Systemrolle
    wert: isolert prosessgruppe med pakket rotfilsystem og eksplisitte ressurs- og I/O-grenser
    href: https://csrc.nist.gov/pubs/sp/800/190/final
  - label: Artefaktstandard
    wert: OCI Image Specification · manifest · konfigurasjon · lag · deskriptorer
    href: https://specs.opencontainers.org/image-spec/
  - label: Kjøretidsstandard
    wert: OCI Runtime Specification · bundle · config.json · livssyklus
    href: https://specs.opencontainers.org/runtime-spec/
  - label: Linux-grunnlag
    wert: namespaces · cgroups · capabilities · LSM · Seccomp
    href: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - label: Windows-grunnlag
    wert: prosess- eller Hyper-V-isolasjon med Windows-containerbilder
    href: https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/
  - label: Persistens
    wert: volum, bind mount eller ekstern tjeneste; skrivbart lag er ingen gjenopprettingsstrategi
    href: https://docs.docker.com/engine/storage/
  - label: Nettverk
    wert: nettverksnamespace/vNIC · bridge/overlay · portpublisering · DNS/tjenesteoppdagelse
    href: https://github.com/containernetworking/cni/blob/main/SPEC.md
  - label: Ressurser
    wert: CPU · minne · PID-er · I/O; vurder grenser og reservasjoner separat
    href: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - label: Orkestrering
    wert: ønsket tilstand, planlegging, omstart, utrulling og tjenesteoppdagelse; ikke en del av bildet
    href: https://kubernetes.io/docs/concepts/workloads/pods/
  - label: Forsyningskjede
    wert: digest · register · proveniens/attestasjon · signatur · policy
    href: https://slsa.dev/spec/v1.2/
  - label: Driftstilstand
    wert: prosess · helse · ressurser · nettverk · lagring · logger · hendelser · ønsket tilstand
    href: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
  - label: Gjenopprettingsobjekt
    wert: distribusjonsdefinisjon, bilder/digester, konfigurasjon, secrets/nøkler og persistente data
    href: https://csrc.nist.gov/pubs/sp/800/190/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - paperless-ngx
  - home-assistant
translationSourceHash: 7091efc5400536d4b1b40e368e564d2ab0d5332766f61e0fdaf976e723bff142
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:34:25.945Z
translationReview: required
---

# Containere: bilder, kjøretid og robust drift

En container er ikke en liten server, og et bilde er ikke en sikkerhetskopi. Teknisk sett kjører en container-runtime vanlige prosesser med et forberedt filsystem, isolerte visninger av kjerne­ressurser og fastsatte ressurs- og sikkerhetsgrenser. Vertskjernen er en del av hver operasjon. Denne arkitekturen gjør applikasjonsmiljøer reproduserbare og raske å erstatte, men flytter ansvaret til bildeopprinnelse, runtime, nettverk, persistens, secrets, ressursstyring og orkestrering.

For administratorer er det viktigste skillet mellom **artefakt**, **kjøretidsinstans** og **driftstilstand**. Et bilde beskriver det oppstartbare filsystemet og metadataene. En container er en konkret kjøring. Databaser, køer, nøkler, konfigurasjon og revisjonsdata har egne livssykluser. Den som behandler disse nivåene samlet som «Docker-containeren», kan verken avgrense en feil eller planlegge en fullstendig gjenoppretting.

Forklaringen begynner med bildet som et leverbart artefakt og følger veien via engine og runtime til kjerne, nettverk og lagring. Orkestrering, sikkerhet, diagnose og gjenoppretting bygger på dette forløpet.

## Teknisk plassering

NIST beskriver applikasjonscontainere som en form for operativsystemvirtualisering: Applikasjoner deler kjernen, mens isolasjon og ressursstyring bygger på operativsystemmekanismer. Virtuelle maskiner virtualiserer derimot maskinvare for en egen gjestekjerne. Containere og VM-er kan kombineres; valget endrer angrepsflate, kompatibilitet, oppstartskostnad og feilområde ([NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)).

En typisk Linux-kjede består av:

1. Registry eller lokal content store med OCI-manifester, konfigurasjoner og lag;
2. Engine eller orkestrator som bestemmer ønsket tilstand, nettverk, monteringer og policy;
3. High-level-runtime som `containerd` eller CRI-O for bilde- og containeradministrasjon;
4. OCI-runtime som `runc`, som oppretter prosessen fra bundle og `config.json`;
5. vertskjerne med namespaces, cgroups, capabilities, Seccomp og en Linux Security Module;
6. containerprosesser som samhandler med miljøet gjennom virtuelle eller monterte filsystemer, sockets og enheter.

Open Container Initiative standardiserer bilde-, runtime- og distribusjonsdeler separat. Dermed kan et OCI-bilde behandles av forskjellige engines og runtimes uten at nettverk, orkestrering eller sikkerhetskopiering automatisk standardiseres ([OCI Image Specification](https://specs.opencontainers.org/image-spec/), [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/), [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-container.svg?v=20260813" title="Interaktive Infografik: Container von OCI-Artefakt und Registry über Engine, Runtime und Kernelisolation bis Netzwerk, Storage, Identität, Supply Chain und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-container.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Bilde: manifest, konfigurasjon og lag

Et OCI-bilde er en rettet innholdsgraf. Et **manifest** viser gjennom deskriptorer og kryptografiske digester til en bildekonfigurasjon og ordnede filsystemlag. En valgfri image index kan samle flere manifester for ulike operativsystemer og arkitekturer. Medietype, digest og størrelse er del av deskriptoren; tagger hører derimot til registerets navneoppløsning og er ikke en uforanderlig identitet ([OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md), [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)).

Lag inneholder filsystemendringer. Ved oppstart settes de sammen til en felles rotvisning gjennom en storage driver og utvides med et skrivbart containerlag. Et slettet secret eller pakkeinnhold kan fortsatt ligge i et tidligere lag. Multi-stage-builds reduserer byggeverktøy og mellomartefakter i det endelige bildet, men erstatter ikke kontroll av secrets og opprinnelse ([Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/), [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)).

### Tagger, digester og plattformvalg

En tagg som `stable`, `3` eller `latest` kan flyttes til en annen manifestdigest. Reproduserbar produksjon fester derfor den godkjente digesten eller dokumenterer minst digesten som ble løst under utrullingen. For multiarkitekturbilder velger klienten en plattform fra indeksen; feil arkitektur, manglende CPU-funksjoner eller emulering kan gi annerledes oppførsel selv med identisk tagg.

En digest svarer på «hvilke byte?», ikke «hvem bygde dem?» eller «er de sikre?». Signaturer og attestasjoner kan binde identitet, byggeopprinnelse og materialer. SLSA beskriver proveniens som etterprøvbar informasjon om hvor, når og hvordan et artefakt ble opprettet; Sigstores `cosign` kan signere og verifisere containerartefakter ([SLSA Specification](https://slsa.dev/spec/v1.2/), [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)).

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

[`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/) viser manifestlister og plattformer, [`docker image inspect`](https://docs.docker.com/reference/cli/docker/image/inspect/) lokale bildemetadata. [`cosign verify`](https://docs.sigstore.dev/cosign/verifying/verify/) kontrollerer signatur og forventet identitet; [`jq`](https://jqlang.org/manual/) formaterer JSON i Unix-eksemplet.

Fra det uforanderlige bildet oppretter runtime den faktisk kjørende containeren. Først da samles skrivbart lag, namespaces, cgroups, monteringer og prosessparametere.

## Runtime, bundle og containerlivssyklus

OCI Runtime Specification beskriver en container som et miljø for en prosess. En **bundle** inneholder `config.json` og et rotfilsystem. Konfigurasjonen fastsetter prosessargumenter, miljø, bruker, monteringer, namespaces, ressurser og ytterligere plattformalternativer. Livssyklusen skiller mellom opprettelse, start, avslutning og sletting; en prosess med status `created` kjører ennå ikke ([OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)).

`runc` er en referanseimplementasjon av dette low-level-grensesnittet. `containerd` administrerer gjennom det bilder, snapshots, containere, tasks og events. Docker Engine tilbyr et høyere API med nettverk, volumer, bygging og betjeningsmodell. Kubernetes kommuniserer på en node via **Container Runtime Interface (CRI)** med en kompatibel runtime; kubelet snakker ikke bare med Docker CLI ([runc](https://github.com/opencontainers/runc), [containerd](https://containerd.io/docs/), [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)).

Disse lagene har separate tilstander. En Kubernetes Pod kan melde `Running` mens en containerprosess er i en omstartsløyfe; en engine kan kjenne en container der runtime-tasken ikke lenger lever; en prosess kan kjøre mens tjenesten ikke tar imot trafikk. Diagnosen begynner derfor med spørsmålet: **Ønsket tilstand, engine-objekt, runtime-task, prosess eller applikasjon?**

## Namespaces: isolert visning, ikke egen maskin

Linux-namespaces isolerer globale ressurser i separate visninger. Kjernemanualsidene nevner blant annet mount-, PID-, network-, IPC-, UTS-, user-, cgroup- og time-namespaces. En prosess kan i hver type namespace bare være medlem av ett namespace; relasjoner kan undersøkes via `/proc/<pid>/ns` ([namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)).

| Namespace | Isolert visning | Administrativ relevans |
|---|---|---|
| Mount | Monteringspunkter og rotfilsystem | Bind mounts, propagation, maskerte vertsstier |
| PID | Prosess-ID-er og hierarki | PID 1, signal- og reaping-atferd |
| Network | Grensesnitt, ruter, porter, brannmurtilstand | Container kan lytte internt uten å være eksternt tilgjengelig |
| UTS | Verts- og domenenavn | kosmetisk identitet, ikke DNS eller sikkerhetsprinsipal |
| IPC | System-V-IPC og POSIX message queues | delt IPC kun med eksplisitt konfigurasjon |
| User | Avbildning av UID-er/GID-er | Container-root kan avbildes uprivilegert utenfor |
| cgroup | Visning av cgroup-hierarki | hindrer ikke ressursbruk alene |
| Time | utvalgte offseter for monotonisk/oppstartstid | sjelden brukt; ingen generell tidssoneisolasjon |

Isolasjon kan konfigureres. Vertsnettverk, vert-PID, `--privileged`, enhetsdeling, brede bind mounts eller ekstra capabilities åpner bevisst grenser. Containernavnet og en intern UID er ikke en sikkerhetsprinsipal på tvers av vert, register og orkestrator.

### PID 1, signaler og ryddig avslutning

I PID-namespacet får den første prosessen særskilte oppgaver. Den mottar signaler annerledes og må samle inn foreldreløse barneprosesser. Shell-wrappere som ikke starter selve programmet med `exec`, kan sluke termineringssignaler. Orkestratorer sender vanligvis først et termineringssignal, venter en grace period og tvinger deretter avslutning. Applikasjonen må stanse nytt arbeid, avslutte løpende operasjoner innen avgrenset tid og flushe tilstand.

Docker kan bruke en liten init-prosess med `--init`; dette reparerer ikke applikasjonen, men gjør signalvideresending og child reaping tydeligere ([Docker run reference – `--init`](https://docs.docker.com/reference/cli/docker/container/run/#init)). For e-postkøer, databaser og indekserere skal maksimal trygg nedstengingstid inngå i distribusjonsmodellen.

## cgroups: styring og regnskapsføring

cgroups grupperer prosesser og bruker kontrollere for ressurser. I cgroup v2 danner prosesser ett enhetlig hierarki. CPU-, minne-, I/O- og PID-kontrollere har forskjellig semantikk: En CPU-grense struper, en minnegrense kan utløse OOM-kill, en PID-grense hindrer nye prosesser eller tråder, og I/O-grenser avhenger av den faktiske blokkenhetsstien ([Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)).

**Request/reservasjon** og **grense** må ikke forveksles. Kubernetes bruker requests til planlegging og limits som kjøretidsgrenser; for minne er grensen reaktiv og håndheves av kjernen under press. En Pod kan evictes eller avsluttes på grunn av node- eller cgroup-press selv om den samlede kapasiteten er tilstrekkelig ([Kubernetes – Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

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

[`docker container inspect`](https://docs.docker.com/reference/cli/docker/container/inspect/) viser konfigurasjon og runtime-tilstand, [`docker container top`](https://docs.docker.com/reference/cli/docker/container/top/) prosessvisningen og [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) løpende ressursverdier. [`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) og [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) kontrollerer vertsprosessen; vert-PID og container-PID kan være ulike.

## Capabilities, Seccomp og Linux Security Modules

Linux deler klassiske root-privilegier i **capabilities**. `CAP_NET_BIND_SERVICE`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` eller `CAP_SYS_PTRACE` åpner for svært forskjellige operasjoner; `CAP_SYS_ADMIN` innebærer særlig omfattende makt. En prosess som ikke kjører som root, kan ha capabilities, og en root-prosess kan få fjernet nesten alle ([capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)).

Seccomp filtrerer systemkall. Docker bruker som standard en profil som blokkerer utvalgte syscalls; `unconfined` fjerner dette laget. AppArmor eller SELinux kan i tillegg begrense objektbasert tilgang. Disse kontrollene utfyller hverandre: En capability tillater en operasjonsklasse, Seccomp kan blokkere det tilhørende systemkallet, og Security Module kan nekte tilgang til et konkret objekt ([Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/), [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)).

`--privileged` er ingen praktisk feilretting. Det utvider enhets- og capability-tilgang og løsner sikkerhetsprofiler. Hvis en applikasjon bare trenger én port, én enkelt enhet eller én skrivebeskyttet sti, frigjøres akkurat denne muligheten og begrunnes i trusselmodellen.

## Root, user namespaces og rootless-drift

UID 0 i containeren er uten user namespace den samme numeriske UID 0 som vertskjernen vurderer. Namespaces begrenser visning og operasjoner, men en kjerne- eller konfigurasjonsfeil rammer fortsatt verten. Et user namespace kan avbilde container-UID-er til uprivilegerte vert-UID-er; rootless-engines kjører daemon og containere uten vert-root ([user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)).

Rootless endrer forutsetningene for nettverk, portbinding, cgroups og lagring, og er derfor en driftsmodell, ikke en universell hardening-bryter. Uavhengig av dette gjelder: Fjern unødvendige capabilities, bruk skrivebeskyttet rotfilsystem, monter skrivbare stier enkeltvis, sett `no-new-privileges` og ikke koble runtime-socket til workloads. Tilgang til Docker- eller CRI-socket er i praksis tilgang til vertens control plane.

Etter prosess- og rettighetsisolasjon kommer tilgjengelighet. Et eget network namespace skaper i utgangspunktet bare en separat visning; ruting, navneoppløsning og publiserte porter må i tillegg bygges opp.

## Nettverk: namespace, CNI og portpublisering

En Linux-container har avhengig av modus sitt eget network namespace. En engine kobler den ofte via et veth-par til en bridge og utfører NAT eller portpublisering. Overlay-nettverk kapsler trafikk mellom verter; service proxies eller eBPF-dataplan fordeler virtuelle tjenesteadresser. CNI standardiserer hvordan en runtime kaller nettverksplugins for å legge til og fjerne en container fra et nettverk; den standardiserer ikke hele nettverksarkitekturen i en klynge ([CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)).

Fire adresser dokumenteres separat:

- **Lytteadresse i prosessen**, for eksempel `127.0.0.1:8080` eller `0.0.0.0:8080` i namespacet;
- **Container-/Pod-IP**, hvis levetid kan være knyttet til instansen;
- **Tjeneste- eller lastbalanseringsadresse** som et mer stabilt tilgangslag;
- **publisert vertadresse og port**, som bestemmer brannmur, NAT og ekstern tilgjengelighet.

`EXPOSE` i Dockerfile publiserer ingen port; den dokumenterer bare tiltenkte porter. Portpublisering er en kjøretidsavgjørelse. En åpen vertsport beviser heller ikke at readiness, TLS eller applikasjonsautentisering fungerer ([Docker – Container networking](https://docs.docker.com/engine/network/)).

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

[`docker network inspect`](https://docs.docker.com/reference/cli/docker/network/inspect/) og [`docker container port`](https://docs.docker.com/reference/cli/docker/container/port/) viser engine-tilordning og publiserte porter. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) henholdsvis [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) viser lyttere; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) tester TCP. For [DNS](/kb/dns) og [TLS](/kb/tls) gjelder deretter egne diagnoseveier.

## Lagring: skrivbart lag, volumer og bind mounts

Det skrivbare containerlaget tilhører instansen. Det egner seg for midlertidige endringer under kjøring, men verken for høy skriveintensiv last eller som varig datalagring. Docker skiller mellom volumer, bind mounts, tmpfs og det skrivbare laget; volumer administreres av engine, mens bind mounts kobler til en konkret vertssti ([Docker – Storage overview](https://docs.docker.com/engine/storage/)).

| Form | Eier av stien | Typisk risiko |
|---|---|---|
| Skrivbart lag | Storage driver/containerinstans | Tap ved erstatning; copy-on-write- og kapasitetskostnader |
| Volum | Engine eller volumeplugin | Navn/plugin/vertstilknytning sikres ikke med bildet |
| Bind mount | Vertadministrator | Vertssti, rettigheter, SELinux-etikett og oppstartsrekkefølge |
| tmpfs | Arbeidsminne | Tap ved stopp; minneforbruk og secret-rester i swap-modellen |
| Ekstern tjeneste | Database, objekt- eller nettverkslagring | Nettverk, identitet, konsistens, kvote og egen gjenopprettingsplan |

En montering dekker over eksisterende filer på målstien. Hvis en forventet nettverksmontering mangler ved oppstart, kan en bind mount-sti ligge på den lokale verten og applikasjonen kan skrive dit uten å merke det. En vellykket startet container beviser derfor ikke at riktig lagringsbackend er aktiv.

Container Storage Interface-standarden definerer et orkestrator-/plugin-grensesnitt for volumer. Den garanterer ikke applikasjonsquiescens, krasjkonsistens eller vellykket gjenopprettingsevne. Databaser trenger fortsatt sine dokumenterte prosedyrer for sikkerhetskopiering og gjenoppretting ([Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [Backup und Disaster Recovery](/kb/backup-dr)).

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

[`docker volume inspect`](https://docs.docker.com/reference/cli/docker/volume/inspect/) leverer volume-driver og mountpoint. [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume), [`Get-PSDrive`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive), [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html) og [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) kontrollerer den faktisk synlige lagringssituasjonen og kapasiteten.

## Konfigurasjon og secrets

Miljøvariabler er praktiske, men er ofte synlige gjennom inspect-utdata, prosessmiljø, krasjrapporter eller supportpakker. Et secret-objekt i en orkestrator er dessuten ikke automatisk kryptert, rotert eller skjult for privilegerte nodeadministratorer. Kubernetes dokumenterer uttrykkelig at secrets som standard lagres ukryptert i etcd dersom Encryption at Rest ikke er konfigurert ([Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

Konfigurasjon inventariseres i fire klasser:

1. ikke-sensitiv, versjonerbar distribusjonskonfigurasjon;
2. secret-referanser og deres eksterne kilde;
3. tilstand generert ved kjøring, som vertsnøkler, databaseskjemaer eller interne CA-er;
4. standardverdier innebygd i bildet, som kan endres ved oppdateringer.

Rotasjon er en tilstandsmaskin: klargjør nytt secret, oppdater eller last på nytt consumeren, dokumenter funksjon, tilbakekall gammelt secret og ta hensyn til cacher og langvarige forbindelser. En containeromstart alene garanterer ikke rotasjon i den eksterne tjenesten.

## Health, readiness, liveness og startup

Prosesstilstand, tjenesteberedskap og forretningsmessig helse er ulike signaler. Docker `HEALTHCHECK` kjører en kommando i containeren og lagrer exitkode og begrenset utdata. Kubernetes skiller mellom startup-, readiness- og liveness-prober: Startup beskytter treg oppstart mot for tidlig liveness, readiness styrer endpoints, og liveness kan utløse en omstart ([Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/#healthcheck), [Kubernetes – Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

En god probe er rimelig, tidsbegrenset og svarer på nøyaktig ett driftsspørsmål. En liveness-probe må ikke utløse omstarter ved hver ekstern delforstyrrelse; ellers forsterker den DNS-, database- eller leverandørproblemer. En readiness-probe kan ta tjenesten ut av trafikken når den ikke trygt kan ta imot nytt arbeid. Grundige ende-til-ende-kontroller hører heller hjemme i overvåking enn i lokale prober hvert sekund.

Compose `depends_on` styrer opprettelses- og oppstartsrekkefølgen; med betingelser kan den vente på health eller en vellykket avsluttet avhengighetsjobb. Den erstatter ikke gjenopprettingslogikk for forbindelser: Tjenester kan også feile uavhengig etter oppstart ([Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)).

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

[`docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/) leser den konfigurerte loggingsbanen, [`docker events`](https://docs.docker.com/reference/cli/docker/system/events/) engine-hendelser og [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) Compose-tilstanden. [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) viser daemonkonteksten under systemd. Docker dokumenterer logging drivers, rotasjon og dual logging separat; ubegrensede JSON-logger kan fylle verten ([Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)).

Healthchecks kan oppdage en feilende prosess, og en orkestrator kan starte den på nytt. Tapte data, feil konfigurasjon eller en ødelagt ekstern avhengighet repareres imidlertid ikke av dette.

## Omstart er ingen gjenopprettingsstrategi

Restart policies reagerer på prosessavslutning. De kjenner verken skadde data, blokkerte køer eller feil credentials. Docker påpeker dessuten at en restart policy først trer i kraft når en container har kjørt vellykket i minst ti sekunder; manuelle stopp undertrykker den frem til daemon-omstart eller manuell start ([Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)).

Omstartsløyfer bruker CPU, genererer logger og kan overbelaste avhengige tjenester. Orkestratorer bruker backoff, men administratoren trenger likevel første feil, exitkode, OOM-/eviction-årsak, siste konfigurasjon og hendelsestidslinje. En CrashLoop er en symptomtilstand, ikke en årsak.

## Orkestrering: Pod, node og ønsket tilstand

Kubernetes grupperer én eller flere containere i en **Pod**. Containere i en Pod deler nettverksnamespace og kan dele volumer; de planlegges samlet. Deployments administrerer ReplicaSets og trinnvise oppdateringer. kubelet implementerer Podspec på noden gjennom CRI, CNI og lagringsplugins ([Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).

| Nivå | Eier | Typisk feil |
|---|---|---|
| Control Plane | API, scheduler, controller | ønsket tilstand beregnes eller planlegges ikke |
| Node/kubelet | lokal implementering | image pull, disk-/PID-/memory pressure, runtime- eller CNI-feil |
| Pod Sandbox | delt Pod-nettverk | opprettelse av sandbox/IP eller tap av namespace |
| Container | bilde og prosess | start-, config-, exit- eller OOM-feil |
| Service/Ingress | tilgjengelighet og ruting | ingen ready endpoints, feil port eller policy |
| Persistent Volume | datasti | attach/mount, sone, rettigheter, snapshot eller backend |

Fasen `Running` betyr at minst én primær container kjører eller starter; den er ingen applikasjons-SLA. Containerstatus, conditions, events, prober og controllertilstand evalueres samlet ([Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)).

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

[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/), [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) og [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) viser API-, hendelses- og containerperspektivet. [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) undersøker CRI-visningen på noden. En `kubectl exec` endrer og observerer kjøretidsinstansen; den erstatter ikke et reproduserbart bilde eller en runbook.

## Compose og deklarative enkeltsystemer på vert

Compose beskriver tjenester, nettverk, volumer, secrets, configs og avhengigheter i en applikasjonsmodell. Den er verdifull for enkelthost- og utviklingsflyter, men er verken registry eller klyngeorkestrator. Compose Specification definerer modellen uavhengig av en bestemt CLI ([Compose Specification](https://compose-spec.io/)).

Et produksjonsklart Compose-arkiv inneholder:

- bildedigester eller kontrollert taggoppløsning og dokumentert plattform;
- eksplisitte monteringer, skrivebeskyttet rotfilsystem og skrivbare stier;
- semantikk for ressurser, omstart, stopp og health;
- separat konfigurasjon og secret-referanser;
- nettverk og publiserte porter;
- spesifikasjoner for logging og rotasjon;
- prosedyrer for backup/gjenoppretting og oppdatering utenfor YAML-filen.

`docker compose config` renderer den sammenslåtte konfigurasjonen og viser variabeloppløsning. Utdata kan inneholde secrets og behandles deretter ([Docker – `docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)).

## Image pull, registry og cache

Et registry distribuerer innhold gjennom manifester og blobs. Autentisering, repository-autorisasjon, taggoppløsning, mirror, proxycache og lokale content stores kan feile hver for seg. Kubernetes `imagePullPolicy` bestemmer når kubelet kontakter registryet; også `Always` bruker lokalt eksisterende lag hvis den løste digesten allerede finnes ([Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)).

En produktiv utrulling protokollerer registry, repository, tagg, løst digest, plattform, beslutning om signatur/proveniens og nodebestand. Garbage collection på registry eller node må ikke fjerne digester som fortsatt trengs for rollback. Air-gapped-drift trenger i tillegg prosess for mirror, nøkler, tilbakekalling og metadata.

## Sikkerhet i forsyningskjede og kjøretid

NIST skiller risikoer i bilder, registre, orkestratorer, containere og vertens operativsystemer. Et scannerresultat er bare ett signal: Pakkeinventaret kan være ufullstendig, en CVE kan være utilgjengelig, eller omvendt kan en applikasjonsspesifikk feil være usynlig. Policy knytter sammen opprinnelse, signatur, kjente sårbarheter, konfigurasjon og kjøretidskontekst ([NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final)).

Kubernetes Pod Security Standards definerer profilene Privileged, Baseline og Restricted. Restricted krever blant annet kjøring som non-root, en Seccomp-profil og sterkt begrensede capabilities; konkrete workloads må likevel funksjonstestes ([Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

Følgende kontrolleres minst:

- pålitelig registry og uforanderlig digest;
- byggeproveniens, signatur og kontrollert nøkkel-/identitetskilde;
- minimalt base image og ingen build secrets i lag;
- non-root, capability-drop, Seccomp/LSM, skrivebeskyttet rotfilsystem, begrensede enheter og monteringer;
- ingen runtime-sockets eller vertsnamespaces uten eksplisitt unntak;
- Network Policies eller vertsbrannmur og kontrollert egress;
- ressurs- og PID-grenser mot lokal uttømming;
- vei for patching, rebuilding, utrulling og rollback.

## Windows-containere

Windows-containere bruker Windows-bilder og Windows-kjernemekanismer. Microsoft skiller mellom **Process Isolation**, der containere deler vertskjernen, og **Hyper-V Isolation**, der hver container kjører i en optimalisert VM med egen kjerne. Begge bruker samme bildeformat og administrasjonsverktøy, men har ulike isolasjons- og kompatibilitetsgrenser ([Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/), [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)).

Ved Process Isolation må verts- og containeroperativsystemene være kompatible. Hyper-V Isolation kan frikoble bestemte versjonsforskjeller, men øker ressurs- og oppstartskostnadene. Microsoft dokumenterer de støttede kombinasjonene av vert og bilde; «Windows Container» er derfor ufullstendig uten opplysninger om base image, build og isolasjon ([Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)).

Linux- og Windows-noder deler ikke samme kjerne og ikke samme bildebinarier. Multi-OS-klynger krever scheduling-etiketter, passende DaemonSets, nettverks-/lagringsplugins og ulike diagnoseveier.

## Oppdateringer, utrulling og rollback

En containeroppdatering er et **bildebytte pluss tilstandsovergang**. Før utrullingen kontrolleres release notes, skjemaendringer, migrasjonstrinn, minimumsversjoner for eksterne tjenester, nye porter/scopes og bakoverkompatibilitet. Rollback av bildet kan være umulig etter en datamigrering som ikke er bakoverkompatibel.

Den kontrollerte prosessen er:

1. Registrer måldigest, signatur, proveniens og scanbeslutning;
2. Opprett en backup eller et testet gjenopprettingspunkt for de persistente dataene;
3. Kontroller konfigurasjons- og skjemadiff;
4. Rull ut canary eller trinnvise replikaer med reelle prober og SLO-er;
5. Overvåk datamigrering og støtte for blandede versjoner;
6. Verifiser digest, instanser, hendelser, feil, latenstid og ressurser;
7. Utløs rollback bare innenfor dokumentert datakompatibilitet.

Denne prosessen knytter sammen [Releases](/kb/releases), [Migration](/kb/migration) og [Backup/DR](/kb/backup-dr). Automatiske taggoppdaterere uten faglige porter flytter bare endringstidspunktet fra endringsprosessen til en bot.

## Backup og disaster recovery

En fullstendig containertjeneste består av mer enn volumer:

- distribusjonsdefinisjoner, policies og nettverksobjekter;
- registrerte bildedigester eller et gjenopprettbart registry-mirror;
- ikke-sensitiv konfigurasjon og secret-/nøkkelkilder;
- persistente data med applikasjonskonsistent metode;
- eksterne databaser, køer, objektlagre, DNS- og identitetsavhengigheter;
- skjemaversjoner, åpne jobber og reconciliation-tilstand;
- runbooks ved tap av node, klynge, registry og lokasjon.

Et tar-arkiv av volumstien kan være inkonsistent med en database som kjører. Lagringssnapshots trenger freeze-/quiesce- eller databasespesifikk semantikk. En gjenopprettingstest bygger tjeneste, nettverk og identiteter på nytt i et rent målmiljø, starter med den sikrede digesten og kontrollerer faglige data samt åpne køer.

Ved feil kontrolleres fra artefakt til prosess og deretter utover: bilde, startparametere, rettigheter, monteringer, navneoppløsning, nettverksvei og eksterne tjenester.

## Diagnose etter avhengighetslag

En containerhendelse avgrenses utenfra og innover:

1. **Ønsket tilstand:** Hvilken definisjon og hvilken digest skal kjøre?
2. **Plassering:** På hvilken vert/node, med hvilken plattform og kapasitet?
3. **Bilde:** Pull, auth, manifest, plattform, signatur og lokalt innhold?
4. **Runtime:** Sandbox, containerstatus, exitkode, OOM, omstart og hendelser?
5. **Prosess:** PID 1, signaler, bruker, capabilities og åpne filer?
6. **Lagring:** Forventet mount, backend, rettigheter, kapasitet og I/O?
7. **Nettverk:** Namespace, DNS, rute, policy, listener, service og TLS?
8. **Applikasjon:** Health, logger, kø, skjema, credential og ekstern avhengighet?

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

[`docker version`](https://docs.docker.com/reference/cli/docker/version/) skiller klient- og serverversjon, [`docker info`](https://docs.docker.com/reference/cli/docker/system/info/) viser engine-, runtime-, lagrings- og sikkerhetskontekst, [`docker context show`](https://docs.docker.com/reference/cli/docker/context/show/) det faktisk administrerte målet og [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) lokalt innholdsforbruk. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html) dokumenterer kjerne og plattform i Unix-eksemplet.

## Teknisk historie

Prosessisolasjon er eldre enn moderne bilder. Unix `chroot` endret en prosess' filsystemrot, men var aldri ment som en fullstendig sikkerhetsgrense. FreeBSD Jails utvidet modellen mot slutten av 1990-årene og med FreeBSD 4.0 med mer isolerte vert- og nettverksvisninger; Solaris Zones kombinerte applikasjonsisolasjon og ressursadministrasjon i operativsystemet ([FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/), [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)).

Linux introduserte gradvis namespaces og Control Groups. LXC kombinerte disse kjernemekanismene til systemcontainere. Docker gjorde fra 2013 bildelag, registrydistribusjon, Dockerfiles og et konsistent utviklergrensesnitt populært; opprinnelig brukte det LXC og byttet senere til et eget runtimebibliotek ([Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/), [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

I 2015 grunnla produsenter og plattformleverandører Open Container Initiative for å standardisere runtime- og bildeformater åpent. `runc` ble OCI-runtime-grunnlaget; `containerd` og CRI-O etablerte high-level-runtimes. Kubernetes abstraherte node-runtimes gjennom CRI, nettverk gjennom CNI og lagring gjennom CSI. I dag utgjør «container» derfor ikke én enkelt produktstack, men en kjede av interoperable spesifikasjoner og implementasjoner ([Open Container Initiative – Overview](https://opencontainers.org/about/overview/), [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)).

## Administrator-sjekkliste på et øyeblikk

Containeren er bare én del av tjenesten. For driftsgodkjenning kontrolleres derfor artefakt, runtime, vert, nettverk, lagring og gjenoppretting som en sammenhengende kjede.

| Spørsmål | Driftsbevis |
|---|---|
| Hvilket artefakt kjører? | Registry, repository, tagg, digest, plattform, configdigest, signatur/proveniens |
| Hvilken runtime-kjede gjelder? | Engine/orkestrator, CRI, high-level-runtime, OCI-runtime, vertskjerne |
| Hvilken isolasjon er aktiv? | Namespaces, Windows-isolasjonsmodus, UID-mapping, capabilities, Seccomp, LSM |
| Hvor ligger tilstand? | Skrivbart lag, volum, bind mount, tmpfs og eksterne tjenester per dataklasse |
| Hva er tilgjengelig? | Lytteadresse, container-/Pod-IP, service, publisert port, ingress og egress-policy |
| Hvem eier identiteten? | Prosess-UID/SID, service account, registry-credential, workload-token og nøkkelkilde |
| Hvilke grenser gjelder? | CPU, minne, PID-er, I/O, disk, loggrotasjon, kvote og node-eviction |
| Hva betyr sunn? | Prosess, startup, readiness, liveness, faglig test og eksterne avhengigheter hver for seg |
| Hvordan endres det? | Godkjent digest, skjema-/configdiff, canary, blandet versjon, rollback-vindu |
| Hvordan gjenopprettes det? | Definisjoner, registry/bilder, secrets/nøkler, data, køer og ren gjenopprettingstest |

## Kilder

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
