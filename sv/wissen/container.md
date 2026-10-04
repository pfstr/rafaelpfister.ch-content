---
title: "Containrar: avbildningar, körtid och robust drift"
blatt: "container"
description: "Containrar för infrastruktur- och meddelandeadministratörer: OCI-artefakter, runtime- och kärnisolering, namnområden och cgroups, nätverk och lagring, orkestrering, identitet och supply chain, resurser, hälsa, loggning, återställning, Windows-containrar och teknisk historia."
fakten:
  - label: Systemroll
    wert: isolerad processmiljö med paketerat rotfilsystem och uttryckliga resurs- och I/O-gränser
    href: https://csrc.nist.gov/pubs/sp/800/190/final
  - label: Artefaktstandard
    wert: OCI Image Specification · manifest · konfiguration · lager · deskriptorer
    href: https://specs.opencontainers.org/image-spec/
  - label: Körtidsstandard
    wert: OCI Runtime Specification · bundle · config.json · livscykel
    href: https://specs.opencontainers.org/runtime-spec/
  - label: Linux-grund
    wert: namnområden · cgroups · capabilities · LSM · Seccomp
    href: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - label: Windows-grund
    wert: Process- eller Hyper-V-isolering med Windows-containeravbildningar
    href: https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/
  - label: Persistens
    wert: volym, bind mount eller extern tjänst; skrivbart lager är ingen återställningsstrategi
    href: https://docs.docker.com/engine/storage/
  - label: Nätverk
    wert: Network Namespace/vNIC · bridge/overlay · portpublicering · DNS/service discovery
    href: https://github.com/containernetworking/cni/blob/main/SPEC.md
  - label: Resurser
    wert: CPU · minne · PID:er · I/O; betrakta gränser och reservationer separat
    href: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - label: Orkestrering
    wert: önskat tillstånd, schemaläggning, omstart, utrullning och tjänsteupptäckt; inte en del av avbildningen
    href: https://kubernetes.io/docs/concepts/workloads/pods/
  - label: Supply chain
    wert: digest · registry · provenance/attestering · signatur · policy
    href: https://slsa.dev/spec/v1.2/
  - label: Drifttillstånd
    wert: process · hälsa · resurser · nätverk · lagring · loggar · händelser · önskat tillstånd
    href: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
  - label: Återställningsobjekt
    wert: distributionsdefinition, avbildningar/digests, konfiguration, secrets/nycklar och persistenta data
    href: https://csrc.nist.gov/pubs/sp/800/190/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - paperless-ngx
  - home-assistant
translationSourceHash: 7091efc5400536d4b1b40e368e564d2ab0d5332766f61e0fdaf976e723bff142
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:31:29.557Z
translationReview: required
---

# Containrar: avbildningar, körtid och robust drift

En container är inte en liten server och en avbildning är ingen säkerhetskopia. Tekniskt kör en containerruntime vanliga processer med ett förberett filsystem, isolerade vyer över kärnresurser och satta resurs- och säkerhetsgränser. Värdkärnan förblir en del av varje operation. Den här arkitekturen gör applikationsmiljöer reproducerbara och snabba att ersätta, men flyttar ansvaret till avbildningens ursprung, runtime, nätverk, persistens, secrets, resursstyrning och orkestrering.

För administratörer är den viktigaste uppdelningen den mellan **artefakt**, **körtidsinstans** och **drifttillstånd**. En avbildning beskriver det startbara filsystemet och metadata. En container är en konkret körning. Databaser, köer, nycklar, konfiguration och revisionsdata har egna livscykler. Den som behandlar dessa nivåer gemensamt som ”Docker-containern” kan varken avgränsa en störning eller planera en återställning fullständigt.

Förklaringen börjar med avbildningen som levererbar artefakt och följer dess väg via motor och runtime till kärna, nätverk och lagring. Orkestrering, säkerhet, diagnostik och återställning bygger på detta förlopp.

## Teknisk klassificering

NIST beskriver applikationscontainrar som en form av operativsystemvirtualisering: Applikationer delar kärnan, medan isolering och resursstyrning bygger på operativsystemmekanismer. Virtuella maskiner virtualiserar däremot hårdvara för en egen gästkärna. Containrar och virtuella maskiner kan kombineras; valet förändrar angreppsyta, kompatibilitet, startkostnad och felområde ([NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)).

En typisk Linux-kedja består av:

1. Registry eller lokal content store med OCI-manifest, konfigurationer och lager;
2. Motor eller orkestrerare som fastställer önskat tillstånd, nätverk, monteringar och policy;
3. High-level-runtime som `containerd` eller CRI-O för hantering av avbildningar och containrar;
4. OCI-runtime som `runc`, som skapar processen från bundle och `config.json`;
5. Värdkärna med namnområden, cgroups, capabilities, Seccomp och ett Linux Security Module;
6. Containerprocesser som interagerar med sin miljö via virtuella eller monterade filsystem, sockets och enheter.

Open Container Initiative standardiserar bild-, runtime- och distributionsdelar separat. Därmed kan en OCI-avbildning bearbetas av olika motorer och runtimes utan att nätverk, orkestrering eller säkerhetskopiering automatiskt standardiseras ([OCI Image Specification](https://specs.opencontainers.org/image-spec/), [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/), [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-container.svg?v=20260813" title="Interaktive Infografik: Container von OCI-Artefakt und Registry über Engine, Runtime und Kernelisolation bis Netzwerk, Storage, Identität, Supply Chain und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-container.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

## Avbildning: manifest, konfiguration och lager

En OCI-avbildning är en riktad innehållsgraf. Ett **manifest** refererar via deskriptorer och kryptografiska digests till en avbildningskonfiguration och ordnade filsystemslager. Ett valfritt image index kan samla flera manifest för olika operativsystem och arkitekturer. Medietyp, digest och storlek ingår i deskriptorn; taggar hör däremot till registry-namnupplösningen och är ingen oföränderlig identitet ([OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md), [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)).

Lager innehåller filsystemsändringar. Vid start sammanfogas de via en storage driver till en gemensam rotvy och kompletteras av ett skrivbart containerlager. Ett raderat secret eller paketinnehåll kan fortfarande finnas i ett tidigare lager. Multi-stage builds minskar byggverktyg och mellanartefakter i den slutliga avbildningen, men ersätter inte kontroll av secrets och ursprung ([Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/), [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)).

### Taggar, digests och plattformsval

En tagg som `stable`, `3` eller `latest` kan flyttas till en annan manifestdigest. Reproducerbar produktion fäster därför den godkända digesten eller dokumenterar åtminstone den digest som upplöstes vid utrullningen. För multi-arch-avbildningar väljer klienten en plattform från indexet; fel arkitektur, saknade CPU-funktioner eller emulering kan ge annat beteende trots identisk tagg.

En digest besvarar ”vilka byte?”, inte ”vem byggde dem?” eller ”är de säkra?”. Signaturer och attesteringar kan binda identitet, byggursprung och material. SLSA beskriver provenance som verifierbar information om var, när och hur en artefakt skapades; Sigstores `cosign` kan signera och verifiera containerartefakter ([SLSA Specification](https://slsa.dev/spec/v1.2/), [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)).

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

[`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/) visar manifestlistor och plattformar, [`docker image inspect`](https://docs.docker.com/reference/cli/docker/image/inspect/) visar lokala avbildningsmetadata. [`cosign verify`](https://docs.sigstore.dev/cosign/verifying/verify/) kontrollerar signatur och förväntad identitet; [`jq`](https://jqlang.org/manual/) formaterar JSON i Unix-exemplet.

Från den oföränderliga avbildningen skapar runtime den container som faktiskt körs. Först då sammanförs skrivbart lager, namnområden, cgroups, monteringar och processparametrar.

## Runtime, bundle och containerlivscykel

OCI Runtime Specification beskriver en container som en miljö för en process. En **bundle** innehåller `config.json` och ett rotfilsystem. Konfigurationen anger processargument, miljö, användare, monteringar, namnområden, resurser och ytterligare plattformsalternativ. Livscykeln skiljer mellan skapande, start, avslut och borttagning; en process med status `created` körs ännu inte ([OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)).

`runc` är en referensimplementation av detta lågnivågränssnitt. `containerd` hanterar ovanpå detta avbildningar, snapshots, containrar, tasks och händelser. Docker Engine erbjuder ett högre API med nätverk, volymer, build- och användningsmodell. Kubernetes kommunicerar på en nod via **Container Runtime Interface (CRI)** med en kompatibel runtime; kubelet talar inte bara med Docker CLI ([runc](https://github.com/opencontainers/runc), [containerd](https://containerd.io/docs/), [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)).

Dessa lager har separata tillstånd. En Kubernetes-pod kan rapportera `Running` medan en containerprocess fastnar i en omstartsslinga; en motor kan känna till en container vars runtime-task inte längre lever; en process kan köras medan tjänsten inte accepterar trafik. Diagnostik börjar därför med frågan: **Önskat tillstånd, motorobjekt, runtime-task, process eller applikation?**

## Namnområden: isolerad vy, ingen egen maskin

Linux-namnområden isolerar globala resurser i separata vyer. Kärnans man-sidor nämner bland annat mount-, PID-, network-, IPC-, UTS-, user-, cgroup- och time-namnområden. En process kan vara medlem i exakt ett namnområde av varje typ; relationer kan undersökas via `/proc/<pid>/ns` ([namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)).

| Namnområde | Isolerad vy | Administrativ relevans |
|---|---|---|
| Mount | Monteringspunkter och rotfilsystem | Bind mounts, propagation, maskerade värdsökvägar |
| PID | Process-ID:n och hierarki | PID 1, signal- och reapingbeteende |
| Network | Gränssnitt, rutter, portar, brandväggstillstånd | Containern kan lyssna internt utan att vara externt nåbar |
| UTS | Värd- och domännamn | Kosmetisk identitet, inte DNS eller säkerhetsprincipal |
| IPC | System-V-IPC och POSIX Message Queues | Delad IPC endast vid uttrycklig konfiguration |
| User | Mappning av UID/GID | Container-root kan mappas till oprivilegierad användare utanför containern |
| cgroup | Vy över cgroup-hierarki | Förhindrar inte ensamt resursanvändning |
| Time | Valda monotona/starttidsoffset | Används sällan; ingen generell tidszonsisolering |

Isolering kan konfigureras. Värdnätverk, värd-PID, `--privileged`, enhetsdelningar, breda bind mounts eller ytterligare capabilities öppnar medvetet gränser. Containernamnet och en intern UID är ingen säkerhetsprincipal över värd, registry och orkestrerare.

### PID 1, signaler och korrekt avslutning

I PID-namnområdet får den första processen särskilda uppgifter. Den tar emot signaler på ett annat sätt och måste samla in föräldralösa barnprocesser. Shell-wrappers som inte startar det egentliga programmet med `exec` kan svälja termineringssignaler. Orkestrerare skickar vanligtvis först en termineringssignal, väntar en grace period och tvingar därefter fram avslut. Applikationen måste stoppa nytt arbete, avsluta pågående operationer inom en begränsad tid och tömma tillstånd.

Docker kan använda en liten init-process med `--init`; detta reparerar inte en applikation, men gör signalvidarebefordran och child reaping mer uttryckliga ([Docker run reference – `--init`](https://docs.docker.com/reference/cli/docker/container/run/#init)). För e-postköer, databaser och indexerare ska maximal säker avstängningstid ingå i distributionsmodellen.

## cgroups: styrning och redovisning

cgroups grupperar processer och tillämpar controllers för resurser. I cgroup v2 bildar processer en enhetlig hierarki. CPU-, minnes-, I/O- och PID-controllers har olika semantik: en CPU-gräns stryper, en minnesgräns kan utlösa en OOM-kill, en PID-gräns förhindrar nya processer eller trådar och I/O-gränser beror på den faktiska blockenhetssökvägen ([Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)).

**Request/reservation** och **limit** får inte förväxlas. Kubernetes använder requests för schemaläggning och limits för körtidsgränser; för minne är gränsen reaktiv och verkställs av kärnan under tryck. En pod kan evictas eller avslutas på grund av node- eller cgroup-tryck trots tillräcklig total kapacitet ([Kubernetes – Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

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

[`docker container inspect`](https://docs.docker.com/reference/cli/docker/container/inspect/) visar konfiguration och runtimetillstånd, [`docker container top`](https://docs.docker.com/reference/cli/docker/container/top/) processvyn och [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) aktuella resursvärden. [`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) och [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) kontrollerar värdprocessen; värd-PID och container-PID kan skilja sig åt.

## Capabilities, Seccomp och Linux Security Modules

Linux delar upp klassiska rootprivilegier i **capabilities**. `CAP_NET_BIND_SERVICE`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` eller `CAP_SYS_PTRACE` öppnar mycket olika operationer; `CAP_SYS_ADMIN` omfattar särskilt omfattande befogenheter. En process som inte körs som root kan ha capabilities och en rootprocess kan få nästan alla borttagna ([capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)).

Seccomp filtrerar systemanrop. Docker använder som standard en profil som blockerar utvalda syscalls; `unconfined` tar bort detta lager. AppArmor eller SELinux kan dessutom begränsa objektrelaterad åtkomst. Dessa kontroller kompletterar varandra: en capability tillåter en operationsklass, Seccomp kan blockera det tillhörande syscall-anropet och Security Module kan neka åtkomst till ett konkret objekt ([Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/), [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)).

`--privileged` är ingen bekväm felåtgärd. Det utökar åtkomst till enheter och capabilities och lättar på säkerhetsprofiler. Om en applikation endast behöver en port, en enskild enhet eller en skrivskyddad sökväg frigörs exakt denna förmåga och motiveras i threat model.

## Root, user namespaces och rootless-drift

UID 0 i containern är utan user namespace samma numeriska UID 0 som värdkärnan bedömer. Namnområden begränsar vy och operationer, men ett kärn- eller konfigurationsfel påverkar fortsatt värden. Ett user namespace kan mappa container-UID:er till oprivilegierade värd-UID:er; rootless-motorer kör daemon och containrar utan root på värden ([user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)).

Rootless förändrar förutsättningarna för nätverk, portbindning, cgroups och lagring och är därför en driftmodell, inte en universell härdningsknapp. Oavsett detta gäller: ta bort onödiga capabilities, kör rotfilsystemet skrivskyddat, montera skrivbara sökvägar var för sig, sätt `no-new-privileges` och montera inte runtime-socket i workloads. Åtkomst till Docker- eller CRI-socket innebär i praktiken åtkomst till värdens control plane.

Efter process- och rättighetsisolering följer nåbarheten. Ett eget network namespace skapar först endast en separat vy; routing, namnupplösning och publicerade portar måste dessutom byggas upp.

## Nätverk: namespace, CNI och portpublicering

En Linux-container har beroende på läge ett eget network namespace. En motor ansluter den ofta via ett veth-par till en bridge och utför NAT eller portpublicering. Overlaynät kapslar in trafik mellan värdar; service proxies eller eBPF-dataplan distribuerar virtuella tjänsteadresser. CNI standardiserar hur en runtime anropar nätverksplugin för att lägga till och ta bort en container från ett nätverk; den standardiserar inte hela nätverksarkitekturen för ett kluster ([CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)).

Fyra adresser dokumenteras separat:

- **Lyssningsadress i processen**, till exempel `127.0.0.1:8080` eller `0.0.0.0:8080` i namnområdet;
- **Container-/pod-IP**, vars livstid kan vara bunden till instansen;
- **Tjänste- eller load balancer-adress** som stabilare åtkomstlager;
- **Publicerad värdadress och port**, som bestämmer brandvägg, NAT och extern nåbarhet.

`EXPOSE` i Dockerfile publicerar ingen port; den dokumenterar endast avsedda portar. Portpublicering är ett körtidsbeslut. En öppen värdport bevisar inte heller att readiness, TLS eller applikationsautentisering fungerar ([Docker – Container networking](https://docs.docker.com/engine/network/)).

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

[`docker network inspect`](https://docs.docker.com/reference/cli/docker/network/inspect/) och [`docker container port`](https://docs.docker.com/reference/cli/docker/container/port/) visar motorns tilldelning och publicerade portar. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) respektive [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) visar lyssnare; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar TCP. För [DNS](/kb/dns) och [TLS](/kb/tls) gäller därefter deras egna diagnostikvägar.

## Lagring: skrivbart lager, volymer och bind mounts

Det skrivbara containerlagret tillhör instansen. Det lämpar sig för tillfälliga körtidsändringar, men varken för hög skrivintensiv last eller som permanent datalagring. Docker skiljer mellan volymer, bind mounts, tmpfs och det skrivbara lagret; volymer hanteras av motorn, bind mounts monterar en konkret värdsökväg ([Docker – Storage overview](https://docs.docker.com/engine/storage/)).

| Form | Ägare till sökvägen | Typisk risk |
|---|---|---|
| Skrivbart lager | Storage driver/containerinstans | Förlust vid ersättning; copy-on-write- och kapacitetskostnader |
| Volym | Motor eller volume-plugin | Namn/plugin/värdbindning säkras inte med avbildningen |
| Bind mount | Värdadministratör | Värdsökväg, rättigheter, SELinux-etikett och startordning |
| tmpfs | Arbetsminne | Förlust vid stopp; minnesförbrukning och secretrester i swap-modellen |
| Extern tjänst | Databas, objekt- eller nätverkslagring | Nätverk, identitet, konsistens, kvot och egen återställningsplan |

En montering täcker över befintliga filer på målsökvägen. Om en förväntad nätverksmontering saknas vid start kan en bind-mount-sökväg ligga på den lokala värden och applikationen obemärkt skriva dit. En framgångsrikt startad container bevisar därför inte att rätt lagringsbackend är aktiv.

Container Storage Interface-standarden definierar ett orkestrerar-/plugingränssnitt för volymer. Den garanterar inte applikationsquiescens, kraschkonsekvens eller lyckad återställningsbarhet. Databaser behöver fortsatt sina dokumenterade säkerhetskopierings- och återställningsmetoder ([Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [Säkerhetskopiering och disaster recovery](/kb/backup-dr)).

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

[`docker volume inspect`](https://docs.docker.com/reference/cli/docker/volume/inspect/) visar volume-driver och monteringspunkt. [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume), [`Get-PSDrive`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive), [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html) och [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) kontrollerar den faktiskt synliga lagringssituationen och kapaciteten.

## Konfiguration och secrets

Miljövariabler är praktiska, men ofta synliga via inspect-utdata, processmiljö, kraschrapporter eller supportpaket. Ett secretobjekt i en orkestrerare är dessutom inte automatiskt krypterat, roterat eller dolt för privilegierade nodadministratörer. Kubernetes dokumenterar uttryckligen att secrets som standard lagras okrypterade i etcd om Encryption at Rest inte har konfigurerats ([Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

Konfiguration inventeras i fyra klasser:

1. icke-känslig, versionshanterbar distributionskonfiguration;
2. secretreferenser och deras externa källa;
3. tillstånd som skapas vid körning, till exempel host keys, databasscheman eller interna CA:er;
4. standardvärden inbyggda i avbildningen, som kan ändras vid uppdateringar.

Rotation är en tillståndsmaskin: tillhandahåll nytt secret, uppdatera eller ladda om konsumenter, bevisa funktion, återkalla gammalt secret och beakta cacher respektive långlivade anslutningar. Enbart en containeromstart garanterar inte rotation i fjärrtjänsten.

## Health, readiness, liveness och startup

Processtillstånd, tjänsteberedskap och verksamhetsmässig hälsa är olika signaler. Docker `HEALTHCHECK` kör ett kommando i containern och lagrar exitkod och begränsad utdata. Kubernetes skiljer mellan startup-, readiness- och liveness-prober: startup skyddar en långsam start från förtida liveness, readiness styr endpoints och liveness kan utlösa en omstart ([Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/#healthcheck), [Kubernetes – Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

En bra prob är billig, tidsbegränsad och besvarar exakt en driftfråga. En liveness-prob får inte utlösa omstarter vid varje extern delstörning; annars förstärker den problem med DNS, databas eller leverantör. En readiness-prob får ta tjänsten ur trafiken när den inte säkert kan acceptera nytt arbete. Djupgående end-to-end-kontroller hör snarare hemma i övervakningen än i lokala prober varje sekund.

Compose `depends_on` styr skapande- och startordning; med villkor kan det vänta på health eller ett avslutat beroendejobb som lyckats. Det ersätter inte återanslutningslogik: tjänster kan också fallera oberoende efter start ([Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)).

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

[`docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/) läser den konfigurerade loggningssökvägen, [`docker events`](https://docs.docker.com/reference/cli/docker/system/events/) motorhändelser och [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) Compose-tillståndet. [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) visar daemonkontexten under systemd. Docker dokumenterar logging drivers, rotation och dual logging separat; obegränsade JSON-loggar kan fylla värden ([Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)).

Healthchecks kan identifiera en felande process och en orkestrerare kan starta om den. Förlorade data, felaktig konfiguration eller ett trasigt externt beroende repareras dock inte av detta.

## Omstart är ingen återställningsstrategi

Restart policies reagerar på processavslut. De känner varken till skadade data, blockerade köer eller felaktiga credentials. Docker påpekar dessutom att en restart policy blir verksam först när en container har körts framgångsrikt i minst tio sekunder; manuella stopp undertrycker den fram till daemonomstart eller manuell start ([Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)).

Omstartsslingor förbrukar CPU, genererar loggar och kan överbelasta beroende tjänster. Orkestrerare använder backoff, men administratören behöver ändå första felet, exitkod, OOM-/evictionorsak, senaste konfiguration och tidslinje för händelser. En CrashLoop är ett symptomtillstånd, ingen orsak.

## Orkestrering: pod, nod och önskat tillstånd

Kubernetes grupperar en eller flera containrar i en **pod**. Containrar i en pod delar nätverksnamnområde och kan dela volymer; de schemaläggs tillsammans. Deployments hanterar ReplicaSets och stegvisa uppdateringar. kubelet verkställer Podspec på noden via CRI, CNI och lagringsplugin ([Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).

| Nivå | Ägare | Typisk störning |
|---|---|---|
| Control plane | API, scheduler, controller | Önskat tillstånd beräknas eller schemaläggs inte |
| Nod/kubelet | Lokal verkställighet | Image pull, disk-/PID-/memory pressure, runtime- eller CNI-fel |
| Pod sandbox | Delat podnätverk | Sandbox-/IP-skapande eller förlust av namespace |
| Container | Avbildning och process | Start-, config-, exit- eller OOM-fel |
| Service/Ingress | Nåbarhet och routing | Inga ready endpoints, fel port eller policy |
| Persistent volume | Datasökväg | Attach/mount, zon, rättigheter, snapshot eller backend |

Fasen `Running` betyder att minst en primär container körs eller startar; den är inget applikations-SLA. Containerstatus, conditions, events, prober och controllertillstånd utvärderas gemensamt ([Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)).

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

[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/), [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) och [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) visar API-, händelse- och containerperspektiv. [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) undersöker CRI-vyn på noden. Ett `kubectl exec` ändrar och observerar körtidsinstansen; det ersätter inte en reproducerbar avbildning eller runbook.

## Compose och deklarativa system på enskilda värdar

Compose beskriver tjänster, nätverk, volymer, secrets, configs och beroenden i en applikationsmodell. Det är värdefullt för enskilda värdar och utvecklingsflöden, men varken registry eller klusterorkestrerare. Compose Specification definierar modellen oberoende av ett visst CLI ([Compose Specification](https://compose-spec.io/)).

En produktionsklar Compose-lagringsplats innehåller:

- image-digests eller kontrollerad taggupplösning och dokumenterad plattform;
- uttryckliga monteringar, skrivskyddat rootfs och skrivbara sökvägar;
- resurs-, restart-, stop- och health-semantik;
- separerade konfigurations- och secretreferenser;
- nätverk och publicerade portar;
- krav på loggning och rotation;
- backup-/restore- och uppdateringsförfaranden utanför YAML-filen.

`docker compose config` renderar den sammanslagna konfigurationen och visar variabelupplösning. Utdata kan innehålla secrets och hanteras därefter ([Docker – `docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)).

## Image pull, registry och cache

Ett registry distribuerar innehåll via manifest och blobs. Autentisering, repositoryauktorisering, taggupplösning, mirror, proxycache och lokala content stores kan misslyckas separat. Kubernetes `imagePullPolicy` avgör när kubelet kontaktar registryt; även `Always` använder lokalt tillgängliga lager om den upplösta digesten redan finns ([Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)).

En produktiv utrullning protokollför registry, repository, tagg, upplöst digest, plattform, signatur-/provenancebeslut och nodinnehåll. Garbage collection på registry eller nod får inte ta bort digests som fortfarande behövs för rollback. Air-gapped-drift behöver dessutom processer för spegel, nycklar, återkallning och metadata.

## Supply-chain- och körtidssäkerhet

NIST skiljer risker i avbildningar, registries, orkestrerare, containrar och värdoperativsystem. Ett skannerresultat är bara en signal: paketinventeringen kan vara ofullständig, en CVE kan vara onåbar eller omvänt kan ett applikationseget fel vara osynligt. Policy förenar ursprung, signatur, kända sårbarheter, konfiguration och körtidskontext ([NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final)).

Kubernetes Pod Security Standards definierar profilerna Privileged, Baseline och Restricted. Restricted kräver bland annat körning som icke-root, en Seccomp-profil och kraftigt begränsade capabilities; konkreta workloads måste ändå funktionstestas ([Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

Minst följande kontrolleras:

- betrodd registry och oföränderlig digest;
- build provenance, signatur och kontrollerad källa för nyckel/identitet;
- minimal base image och inga build secrets i lager;
- non-root, capability drop, Seccomp/LSM, skrivskyddat rootfs, begränsade enheter och monteringar;
- inga runtime-sockets eller host namespaces utan uttryckligt undantag;
- network policies respektive värdbrandvägg och kontrollerad egress;
- resurs- och PID-gränser mot lokal utmattning;
- sökväg för patch, rebuild, rollout och rollback.

## Windows-containrar

Windows-containrar använder Windows-avbildningar och Windows-kärnmekanismer. Microsoft skiljer mellan **Process Isolation**, där containrar delar värdkärnan, och **Hyper-V Isolation**, där varje container körs i en optimerad virtuell maskin med egen kärna. Båda använder samma avbildningsformat och administrationsverktyg, men har olika isolerings- och kompatibilitetsgränser ([Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/), [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)).

Vid Process Isolation måste värd- och containeroperativsystem vara kompatibla. Hyper-V Isolation kan frikoppla vissa versionsskillnader, men ökar resurs- och startkostnader. Microsoft dokumenterar de kombinationer av värd och avbildning som stöds; ”Windows Container” är därför ofullständigt utan uppgift om base image, build och isolering ([Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)).

Linux- och Windows-noder delar varken samma kärna eller samma image-binära filer. Multi-OS-kluster behöver schemaläggningsetiketter, lämpliga DaemonSets, nätverks-/lagringsplugin och olika diagnostikvägar.

## Uppdateringar, rollout och rollback

En containeruppdatering är ett **avbildningsbyte plus tillståndsövergång**. Före utrullningen kontrolleras release notes, schemaändringar, migreringssteg, minimiversioner för externa tjänster, nya portar/scopes och bakåtkompatibilitet. En rollback av avbildningen kan vara omöjlig efter en icke bakåtkompatibel datamigrering.

Det kontrollerade förloppet är:

1. Dokumentera måldigest, signatur, provenance och skanningsbeslut;
2. Skapa en backup eller testad återställningspunkt för persistenta data;
3. Kontrollera konfigurations- och schemadiff;
4. Rulla ut canary eller stegvisa repliker med verkliga prober och SLO:er;
5. Övervaka datamigrering och förmåga till mixed version;
6. Verifiera digest, instanser, händelser, fel, latens och resurser;
7. Utlös rollback endast inom dokumenterad datakompatibilitet.

Denna process förenar [Releaser](/kb/releases), [Migrering](/kb/migration) och [Backup/DR](/kb/backup-dr). Automatiska tagguppdaterare utan verksamhetsmässiga gates flyttar bara ändringstidpunkten från changeprocessen till en bot.

## Säkerhetskopiering och disaster recovery

En fullständig containertjänst består av mer än volymer:

- distributionsdefinitioner, policyer och nätverksobjekt;
- registrerade image-digests eller en återställbar registry-spegel;
- icke-känslig konfiguration samt secret-/nyckelkällor;
- persistenta data med applikationskonsistent metod;
- externa databaser, köer, objektlager, DNS- och identitetsberoenden;
- schemaversioner, öppna jobb och reconciliationtillstånd;
- runbooks för förlust av nod, kluster, registry och plats.

Ett tar-arkiv av volymsökvägen kan vara inkonsekvent med en körande databas. Lagringssnapshots kräver freeze-/quiesce- eller databasspecifik semantik. Ett återställningstest bygger upp tjänst, nätverk och identiteter på nytt i en ren målmiljö, startar med den säkrade digesten och kontrollerar verksamhetsdata samt öppna köer.

Vid störningar kontrolleras från artefakten till processen och sedan utåt: avbildning, startparametrar, rättigheter, monteringar, namnupplösning, nätverksväg och externa tjänster.

## Diagnostik enligt beroendelager

En containerincident avgränsas utifrån och in:

1. **Önskat tillstånd:** Vilken definition och vilken digest ska köras?
2. **Placering:** På vilken värd/nod, med vilken plattform och kapacitet?
3. **Avbildning:** Pull, auth, manifest, plattform, signatur och lokalt innehåll?
4. **Runtime:** Sandbox, containerstatus, exitkod, OOM, omstart och händelser?
5. **Process:** PID 1, signaler, användare, capabilities och öppna filer?
6. **Lagring:** Förväntad montering, backend, rättigheter, kapacitet och I/O?
7. **Nätverk:** Namespace, DNS, rutt, policy, lyssnare, tjänst och TLS?
8. **Applikation:** Health, loggar, kö, schema, credential och externt beroende?

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

[`docker version`](https://docs.docker.com/reference/cli/docker/version/) skiljer mellan klient- och serverversion, [`docker info`](https://docs.docker.com/reference/cli/docker/system/info/) visar motor-, runtime-, lagrings- och säkerhetskontext, [`docker context show`](https://docs.docker.com/reference/cli/docker/context/show/) det faktiskt administrerade målet och [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) lokal innehållsförbrukning. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html) dokumenterar kärna och plattform i Unix-exemplet.

## Teknisk historia

Processisolering är äldre än moderna avbildningar. Unix `chroot` ändrade en process filsystemrot, men var aldrig avsett som en fullständig säkerhetsgräns. FreeBSD Jails utökade modellen i slutet av 1990-talet respektive med FreeBSD 4.0 med mer isolerade värd- och nätverksvyer; Solaris Zones kombinerade applikationsisolering och resurshantering i operativsystemet ([FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/), [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)).

Linux införde successivt namnområden och Control Groups. LXC kombinerade dessa kärnmekanismer till systemcontainrar. Docker gjorde från 2013 image-lager, registrydistribution, Dockerfiles och ett konsekvent utvecklargränssnitt populära; inledningsvis använde det LXC och bytte senare till ett eget runtime-bibliotek ([Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/), [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

År 2015 grundade tillverkare och plattformsleverantörer Open Container Initiative för att öppet standardisera runtime- och imageformat. `runc` blev OCI:s runtimegrund; `containerd` och CRI-O etablerade high-level-runtimes. Kubernetes abstraherade nodruntimes via CRI, nätverk via CNI och lagring via CSI. I dag utgör ”container” därför inte en enskild produktstack, utan en kedja av interoperabla specifikationer och implementationer ([Open Container Initiative – Overview](https://opencontainers.org/about/overview/), [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)).

## Adminchecklista i korthet

Containern är bara en del av tjänsten. För driftgodkännande kontrolleras därför artefakt, runtime, värd, nätverk, lagring och återställning som en sammanhängande kedja.

| Fråga | Driftbevis |
|---|---|
| Vilken artefakt körs? | Registry, repository, tagg, digest, plattform, configdigest, signatur/provenance |
| Vilken runtimekedja gäller? | Motor/orkestrerare, CRI, high-level-runtime, OCI-runtime, värdkärna |
| Vilken isolering är aktiv? | Namnområden, Windows-isoleringsläge, UID-mappning, capabilities, Seccomp, LSM |
| Var finns tillståndet? | Skrivbart lager, volym, bind mount, tmpfs och externa tjänster per dataklass |
| Vad är nåbart? | Lyssningsadress, container-/pod-IP, service, publicerad port, ingress och egresspolicy |
| Vem äger identiteten? | Process-UID/SID, service account, registrycredential, workloadtoken och nyckelkälla |
| Vilka gränser gäller? | CPU, minne, PID:er, I/O, disk, loggrotation, kvot och node eviction |
| Vad betyder frisk? | Process, startup, readiness, liveness, verksamhetstest och externa beroenden separat |
| Hur ändras det? | Godkänd digest, schema-/configdiff, canary, mixed version, rollbackfönster |
| Hur återställs det? | Definitioner, registry/images, secrets/keys, data, köer och rent återställningstest |

## Källor

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
