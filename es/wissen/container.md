---
title: "Contenedores: imágenes, tiempo de ejecución y operación fiable"
blatt: "container"
description: "Contenedores para administradores de infraestructura y mensajería: artefactos OCI, aislamiento del tiempo de ejecución y del kernel, espacios de nombres y cgroups, red y almacenamiento, orquestación, identidad y cadena de suministro, recursos, salud, registro, recuperación, contenedores de Windows e historia técnica."
fakten:
  - label: Función del sistema
    wert: conjunto de procesos aislados con sistema de archivos raíz empaquetado y límites explícitos de recursos y E/S
    href: https://csrc.nist.gov/pubs/sp/800/190/final
  - label: Estándar de artefactos
    wert: OCI Image Specification · manifiesto · configuración · capas · descriptores
    href: https://specs.opencontainers.org/image-spec/
  - label: Estándar de tiempo de ejecución
    wert: OCI Runtime Specification · bundle · config.json · ciclo de vida
    href: https://specs.opencontainers.org/runtime-spec/
  - label: Base Linux
    wert: espacios de nombres · cgroups · capabilities · LSM · Seccomp
    href: https://man7.org/linux/man-pages/man7/namespaces.7.html
  - label: Base Windows
    wert: aislamiento de procesos o Hyper-V con imágenes de contenedores de Windows
    href: https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/
  - label: Persistencia
    wert: volumen, Bind Mount o servicio externo; Writable Layer no es una estrategia de recuperación
    href: https://docs.docker.com/engine/storage/
  - label: Red
    wert: Network Namespace/vNIC · Bridge/Overlay · publicación de puertos · DNS/Service Discovery
    href: https://github.com/containernetworking/cni/blob/main/SPEC.md
  - label: Recursos
    wert: CPU · memoria · PID · E/S; considerar por separado límites y reservas
    href: https://docs.kernel.org/admin-guide/cgroup-v2.html
  - label: Orquestación
    wert: estado deseado, scheduling, reinicio, despliegue y descubrimiento de servicios; no forma parte de la imagen
    href: https://kubernetes.io/docs/concepts/workloads/pods/
  - label: Cadena de suministro
    wert: digest · registro · provenance/attestation · firma · política
    href: https://slsa.dev/spec/v1.2/
  - label: Estado operativo
    wert: proceso · salud · recursos · red · almacenamiento · logs · eventos · estado deseado
    href: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
  - label: Objeto de recuperación
    wert: definición de despliegue, imágenes/digests, configuración, secretos/claves y datos persistentes
    href: https://csrc.nist.gov/pubs/sp/800/190/final
werbung:
  - newsletter
ctaThemen:
  - rclone
  - paperless-ngx
  - home-assistant
translationSourceHash: 7091efc5400536d4b1b40e368e564d2ab0d5332766f61e0fdaf976e723bff142
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:30:00.515Z
translationReview: required
---

# Contenedores: imágenes, tiempo de ejecución y operación fiable

Un contenedor no es un servidor pequeño ni una imagen es una copia de seguridad. Técnicamente, un tiempo de ejecución de contenedores ejecuta procesos convencionales con un sistema de archivos preparado, vistas aisladas de los recursos del kernel y límites de recursos y seguridad establecidos. El kernel del host forma parte de cada operación. Esta arquitectura hace que los entornos de aplicación sean reproducibles y reemplazables con rapidez; sin embargo, desplaza la responsabilidad al origen de la imagen, el runtime, la red, la persistencia, los secretos, el control de recursos y la orquestación.

Para los administradores, la separación más importante es la existente entre **artefacto**, **instancia de ejecución** y **estado operativo**. Una imagen describe el sistema de archivos iniciable y los metadatos. Un contenedor es una ejecución concreta. Bases de datos, colas, claves, configuración y datos de auditoría tienen sus propios ciclos de vida. Quien trate estos niveles conjuntamente como «el contenedor Docker» no podrá ni acotar una incidencia ni planificar completamente una restauración.

La explicación comienza con la imagen como artefacto distribuible y sigue su recorrido a través del motor y el runtime hasta el kernel, la red y el almacenamiento. La orquestación, la seguridad, el diagnóstico y la recuperación se basan en este flujo.

## Clasificación técnica

NIST describe los contenedores de aplicaciones como una forma de virtualización del sistema operativo: las aplicaciones comparten el kernel, mientras que el aislamiento y el control de recursos se basan en mecanismos del sistema operativo. En cambio, las máquinas virtuales virtualizan hardware para su propio kernel invitado. Los contenedores y las VM pueden combinarse; la elección modifica la superficie de ataque, la compatibilidad, el coste de inicio y el ámbito de fallo ([NIST SP 800-190 – Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)).

Una ruta típica de Linux consta de:

1. un registro o almacén de contenido local con manifiestos, configuraciones y capas OCI;
2. un motor u orquestador que determina el estado deseado, la red, los montajes y la política;
3. un runtime de alto nivel como `containerd` o CRI-O para la gestión de imágenes y contenedores;
4. un runtime OCI como `runc`, que crea el proceso a partir del bundle y `config.json`;
5. el kernel del host con espacios de nombres, cgroups, capabilities, Seccomp y un Linux Security Module;
6. procesos de contenedor que interactúan con su entorno mediante sistemas de archivos virtuales o montados, sockets y dispositivos.

La Open Container Initiative estandariza por separado las partes de imagen, runtime y distribución. Así, diferentes motores y runtimes pueden procesar una imagen OCI sin que por ello queden estandarizadas automáticamente la red, la orquestación o la copia de seguridad ([OCI Image Specification](https://specs.opencontainers.org/image-spec/), [OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/), [OCI Distribution Specification](https://specs.opencontainers.org/distribution-spec/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-container.svg?v=20260813" title="Interaktive Infografik: Container von OCI-Artefakt und Registry über Engine, Runtime und Kernelisolation bis Netzwerk, Storage, Identität, Supply Chain und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-container.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Imagen: manifiesto, configuración y capas

Una imagen OCI es un grafo de contenido dirigido. Un **manifiesto** referencia, mediante descriptores y digests criptográficos, una configuración de imagen y capas de sistema de archivos ordenadas. Un índice de imagen opcional puede agrupar varios manifiestos para distintos sistemas operativos y arquitecturas. El tipo de medio, el digest y el tamaño forman parte del descriptor; las etiquetas, en cambio, pertenecen a la resolución de nombres del registro y no son una identidad inmutable ([OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md), [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)).

Las capas contienen cambios en el sistema de archivos. Al iniciar, se ensamblan mediante un controlador de almacenamiento en una vista raíz común y se complementan con una capa de contenedor escribible. Un secreto o contenido de paquete eliminado puede seguir existiendo en una capa anterior. Los builds multietapa reducen las herramientas de compilación y los artefactos intermedios en la imagen final, pero no sustituyen la comprobación de secretos y procedencia ([Docker – Storage drivers](https://docs.docker.com/engine/storage/drivers/), [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)).

### Etiquetas, digests y selección de plataforma

Una etiqueta como `stable`, `3` o `latest` puede moverse para apuntar a otro digest de manifiesto. Por ello, una producción reproducible fija el digest aprobado o documenta, como mínimo, el digest resuelto durante el despliegue. En imágenes multiarquitectura, el cliente selecciona una plataforma del índice; una arquitectura incorrecta, características de CPU ausentes o la emulación pueden provocar un comportamiento diferente pese a usar la misma etiqueta.

Un digest responde a «¿qué bytes?», no a «¿quién los compiló?» ni «¿son seguros?». Las firmas y las attestations pueden vincular identidad, procedencia de compilación y materiales. SLSA describe la provenance como información verificable sobre dónde, cuándo y cómo se generó un artefacto; `cosign` de Sigstore puede firmar y verificar artefactos de contenedor ([SLSA Specification](https://slsa.dev/spec/v1.2/), [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)).

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

[`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/) muestra listas de manifiestos y plataformas, [`docker image inspect`](https://docs.docker.com/reference/cli/docker/image/inspect/) muestra metadatos locales de imágenes. [`cosign verify`](https://docs.sigstore.dev/cosign/verifying/verify/) verifica la firma y la identidad esperada; [`jq`](https://jqlang.org/manual/) formatea JSON en el ejemplo Unix.

A partir de la imagen inmutable, el runtime crea el contenedor que realmente se ejecuta. Solo entonces se combinan la capa escribible, los espacios de nombres, los cgroups, los montajes y los parámetros de proceso.

## Runtime, bundle y ciclo de vida del contenedor

La OCI Runtime Specification describe un contenedor como un entorno para un proceso. Un **bundle** contiene `config.json` y un sistema de archivos raíz. La configuración establece argumentos del proceso, entorno, usuario, montajes, espacios de nombres, recursos y otras opciones de plataforma. El ciclo de vida distingue creación, inicio, finalización y eliminación; un proceso con estado `created` aún no se ejecuta ([OCI Runtime Specification](https://specs.opencontainers.org/runtime-spec/)).

`runc` es una implementación de referencia de esta interfaz de bajo nivel. `containerd` gestiona por encima de ella imágenes, snapshots, contenedores, tareas y eventos. Docker Engine ofrece una API de nivel superior con red, volúmenes, modelo de compilación y uso. En un nodo, Kubernetes se comunica mediante la **Container Runtime Interface (CRI)** con un runtime compatible; kubelet no habla simplemente con la CLI de Docker ([runc](https://github.com/opencontainers/runc), [containerd](https://containerd.io/docs/), [Kubernetes – Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)).

Estas capas tienen estados separados. Un pod de Kubernetes puede informar `Running` mientras un proceso de contenedor está en un bucle de reinicios; un motor puede conocer un contenedor cuya tarea de runtime ya no está viva; un proceso puede ejecutarse mientras el servicio no acepta tráfico. Por ello, el diagnóstico empieza con la pregunta: **¿estado deseado, objeto del motor, tarea de runtime, proceso o aplicación?**

## Espacios de nombres: vista aislada, no una máquina propia

Los espacios de nombres de Linux aíslan recursos globales en vistas separadas. Las páginas de manual del kernel mencionan, entre otros, espacios de nombres de montaje, PID, red, IPC, UTS, usuario, cgroup y tiempo. Un proceso puede ser miembro de exactamente un espacio de nombres de cada tipo; las relaciones se pueden examinar mediante `/proc/<pid>/ns` ([namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)).

| Espacio de nombres | Vista aislada | Relevancia administrativa |
|---|---|---|
| Mount | Puntos de montaje y sistema de archivos raíz | Bind Mounts, propagación, rutas de host enmascaradas |
| PID | ID de procesos y jerarquía | PID 1, comportamiento de señales y recolección |
| Network | Interfaces, rutas, puertos y estado de firewall | El contenedor puede escuchar internamente sin ser accesible externamente |
| UTS | Nombre de host y de dominio | Identidad cosmética, no DNS ni principal de seguridad |
| IPC | IPC System V y colas de mensajes POSIX | IPC compartido solo con configuración explícita |
| User | Asignación de UID/GID | El root del contenedor puede mapearse sin privilegios fuera |
| cgroup | Vista de la jerarquía de cgroups | Por sí solo no impide el uso de recursos |
| Time | Desplazamientos seleccionados de tiempo monótono/de arranque | Poco utilizado; sin aislamiento general de zona horaria |

El aislamiento es configurable. La red del host, el PID del host, `--privileged`, las autorizaciones de dispositivos, los Bind Mounts amplios o capabilities adicionales abren límites de forma deliberada. El nombre del contenedor y una UID interna no son un principal de seguridad a través del host, el registro y el orquestador.

### PID 1, señales y finalización limpia

En el espacio de nombres PID, el primer proceso asume funciones especiales. Recibe señales de manera diferente y debe recoger procesos hijo huérfanos. Los wrappers de shell que no inician el programa real con `exec` pueden perder las señales de terminación. Los orquestadores suelen enviar primero una señal de terminación, esperar un periodo de gracia y después forzar la finalización. La aplicación debe dejar de aceptar trabajo nuevo, completar de forma limitada las operaciones en curso y vaciar los estados.

Docker puede utilizar un pequeño proceso init con `--init`; esto no corrige una aplicación, pero hace más explícitos el reenvío de señales y la recolección de hijos ([Docker run reference – `--init`](https://docs.docker.com/reference/cli/docker/container/run/#init)). En colas de correo, bases de datos e indexadores, el tiempo máximo de apagado seguro forma parte del modelo de despliegue.

## cgroups: control y contabilidad

Los cgroups agrupan procesos y aplican controladores de recursos. En cgroup v2, los procesos forman una jerarquía unificada. Los controladores de CPU, memoria, E/S y PID tienen semánticas diferentes: un límite de CPU limita la velocidad, un límite de memoria puede provocar un OOM kill, un límite de PID impide nuevos procesos o hilos, y los límites de E/S dependen de la ruta real del dispositivo de bloques ([Linux kernel – Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)).

No deben confundirse **request/reservation** y **limit**. Kubernetes utiliza requests para la programación y limits para los límites de ejecución; para la memoria, el límite es reactivo y el kernel lo aplica bajo presión. Un pod puede ser desalojado o terminado por presión del nodo o del cgroup pese a existir suficiente capacidad total ([Kubernetes – Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

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

[`docker container inspect`](https://docs.docker.com/reference/cli/docker/container/inspect/) muestra la configuración y el estado de runtime, [`docker container top`](https://docs.docker.com/reference/cli/docker/container/top/) la vista de procesos y [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) los valores de recursos en ejecución. [`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) y [`ps`](https://man7.org/linux/man-pages/man1/ps.1.html) comprueban el proceso del host; el PID del host y el PID del contenedor pueden ser distintos.

## Capabilities, Seccomp y Linux Security Modules

Linux divide los privilegios root clásicos en **capabilities**. `CAP_NET_BIND_SERVICE`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` o `CAP_SYS_PTRACE` habilitan operaciones muy diferentes; `CAP_SYS_ADMIN` abarca un poder especialmente amplio. Un proceso que no se ejecuta como root puede tener capabilities, y a un proceso root se le pueden quitar casi todas ([capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)).

Seccomp filtra llamadas al sistema. Docker utiliza por defecto un perfil que bloquea syscalls seleccionadas; `unconfined` elimina esta capa. AppArmor o SELinux pueden limitar adicionalmente accesos relacionados con objetos. Estos controles se complementan: una capability permite una clase de operaciones, Seccomp puede bloquear la syscall correspondiente y el Security Module puede denegar el acceso a un objeto concreto ([Docker – Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/), [Docker – AppArmor security profiles](https://docs.docker.com/engine/security/apparmor/)).

`--privileged` no es una solución cómoda de problemas. Amplía el acceso a dispositivos y capabilities, y relaja perfiles de seguridad. Si una aplicación solo necesita un puerto, un único dispositivo o una ruta de solo lectura, se habilita exactamente esa capacidad y se justifica en el modelo de amenazas.

## Root, espacios de nombres de usuario y operación rootless

La UID 0 dentro del contenedor es, sin un espacio de nombres de usuario, la misma UID numérica 0 que evalúa el kernel del host. Los espacios de nombres limitan la vista y las operaciones, pero un error de kernel o configuración sigue afectando al host. Un espacio de nombres de usuario puede mapear las UID del contenedor a UID sin privilegios del host; los motores rootless ejecutan daemon y contenedores sin root en el host ([user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), [Docker – Rootless mode](https://docs.docker.com/engine/security/rootless/)).

Rootless modifica los requisitos de red, vinculación de puertos, cgroups y almacenamiento, por lo que es un modelo operativo, no un interruptor universal de endurecimiento. Independientemente de ello: eliminar capabilities innecesarias, operar el sistema de archivos raíz en solo lectura, montar por separado rutas escribibles, establecer `no-new-privileges` y no conectar el socket del runtime a workloads. El acceso al socket de Docker o CRI equivale de facto al acceso al plano de control del host.

Tras el aislamiento de procesos y permisos sigue la accesibilidad. Un espacio de nombres de red propio solo crea inicialmente una vista separada; el enrutamiento, la resolución de nombres y los puertos publicados deben configurarse adicionalmente.

## Red: espacio de nombres, CNI y publicación de puertos

Un contenedor Linux posee, según el modo, su propio espacio de nombres de red. Un motor lo conecta con frecuencia mediante un par veth a un bridge y realiza NAT o publicación de puertos. Las redes overlay encapsulan tráfico entre hosts; los proxies de servicio o las rutas de datos eBPF distribuyen direcciones de servicio virtuales. CNI estandariza cómo un runtime llama a plugins de red para añadir y quitar un contenedor de una red; no estandariza toda la arquitectura de red de un clúster ([CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)).

Se documentan por separado cuatro direcciones:

- **dirección de escucha en el proceso**, por ejemplo `127.0.0.1:8080` o `0.0.0.0:8080` en el espacio de nombres;
- **IP de contenedor/pod**, cuya duración puede estar vinculada a la instancia;
- **dirección de servicio o balanceador de carga** como capa de acceso más estable;
- **dirección y puerto de host publicados**, que determinan firewall, NAT y accesibilidad externa.

`EXPOSE` en el Dockerfile no publica un puerto; simplemente documenta los puertos previstos. La publicación de puertos es una decisión de runtime. Del mismo modo, un puerto abierto en el host no prueba que funcionen la readiness, TLS o la autenticación de la aplicación ([Docker – Container networking](https://docs.docker.com/engine/network/)).

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

[`docker network inspect`](https://docs.docker.com/reference/cli/docker/network/inspect/) y [`docker container port`](https://docs.docker.com/reference/cli/docker/container/port/) muestran la asignación del motor y los puertos publicados. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran listeners; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban TCP. Para [DNS](/kb/dns) y [TLS](/kb/tls) se aplican posteriormente sus propias rutas de diagnóstico.

## Almacenamiento: capa escribible, volúmenes y Bind Mounts

La capa de contenedor escribible pertenece a la instancia. Es adecuada para cambios temporales de runtime, pero no para cargas de escritura intensiva ni como almacenamiento permanente de datos. Docker distingue entre volúmenes, Bind Mounts, tmpfs y la Writable Layer; los volúmenes son gestionados por el motor, mientras que los Bind Mounts montan una ruta concreta del host ([Docker – Storage overview](https://docs.docker.com/engine/storage/)).

| Forma | Propietario de la ruta | Riesgo típico |
|---|---|---|
| Writable Layer | Controlador de almacenamiento/instancia de contenedor | Pérdida al sustituir; costes de copy-on-write y capacidad |
| Volume | Motor o plugin de volumen | El nombre, plugin o vínculo con el host no se guarda con la imagen |
| Bind Mount | Administrador del host | Ruta del host, permisos, etiqueta SELinux y orden de inicio |
| tmpfs | Memoria | Pérdida al detenerse; consumo de memoria y restos de secretos en el modelo de swap |
| Servicio externo | Base de datos, almacenamiento de objetos o de red | Red, identidad, consistencia, cuota y plan de recuperación propio |

Un montaje oculta los archivos existentes en la ruta de destino. Si falta un montaje de red esperado al iniciar, una ruta de Bind Mount puede encontrarse en el host local y la aplicación puede escribir ahí sin advertirlo. Por tanto, un contenedor iniciado correctamente no prueba que esté activo el backend de almacenamiento correcto.

El estándar Container Storage Interface define una interfaz entre orquestador y plugin para volúmenes. No garantiza la quiescencia de la aplicación, la consistencia ante fallos ni la capacidad de restauración correcta. Las bases de datos siguen necesitando sus procedimientos documentados de copia de seguridad y restauración ([Container Storage Interface Specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [Copia de seguridad y recuperación ante desastres](/kb/backup-dr)).

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

[`docker volume inspect`](https://docs.docker.com/reference/cli/docker/volume/inspect/) proporciona el controlador de volumen y el punto de montaje. [`Get-Volume`](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume), [`Get-PSDrive`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-psdrive), [`findmnt`](https://man7.org/linux/man-pages/man8/findmnt.8.html) y [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) comprueban la situación de almacenamiento y la capacidad realmente visibles.

## Configuración y secretos

Las variables de entorno son prácticas, pero a menudo son visibles mediante salidas de inspect, el entorno del proceso, informes de fallo o paquetes de soporte. Además, un objeto secreto en un orquestador no está automáticamente cifrado, rotado ni oculto para administradores privilegiados de nodos. Kubernetes documenta expresamente que los secretos se almacenan por defecto sin cifrar en etcd si no se configura el cifrado en reposo ([Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)).

La configuración se inventaría en cuatro clases:

1. configuración de despliegue no sensible y versionable;
2. referencias de secretos y su fuente externa;
3. estado generado durante el runtime, como claves de host, esquemas de bases de datos o CA internas;
4. valores predeterminados incorporados en la imagen, que pueden cambiar con las actualizaciones.

La rotación es una máquina de estados: proporcionar un secreto nuevo, actualizar o recargar consumidores, demostrar el funcionamiento, revocar el secreto antiguo y tener en cuenta cachés o conexiones duraderas. Un reinicio de contenedor por sí solo no garantiza la rotación en el servicio remoto.

## Salud, readiness, liveness y arranque

El estado del proceso, la disponibilidad del servicio y la salud funcional son señales diferentes. Docker `HEALTHCHECK` ejecuta un comando en el contenedor y guarda el código de salida y una salida limitada. Kubernetes distingue entre probes de startup, readiness y liveness: startup protege los arranques lentos de una liveness prematura, readiness controla endpoints y liveness puede desencadenar un reinicio ([Dockerfile reference – HEALTHCHECK](https://docs.docker.com/reference/dockerfile/#healthcheck), [Kubernetes – Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)).

Una buena probe es económica, limitada en el tiempo y responde exactamente a una pregunta operativa. Una liveness probe no debe desencadenar reinicios ante cada fallo parcial externo; de lo contrario, agrava problemas de DNS, base de datos o proveedor. Una readiness probe puede retirar el servicio del tráfico cuando no puede aceptar nuevo trabajo de forma segura. Las comprobaciones exhaustivas de extremo a extremo pertenecen más a la monitorización que a probes locales ejecutadas cada segundo.

Compose `depends_on` controla el orden de creación e inicio; con condiciones puede esperar a la salud o a un trabajo de dependencia finalizado con éxito. No sustituye la lógica de reconexión: los servicios también pueden fallar independientemente después del inicio ([Docker Compose – Control startup order](https://docs.docker.com/compose/how-tos/startup-order/)).

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

[`docker container logs`](https://docs.docker.com/reference/cli/docker/container/logs/) lee la ruta de logging configurada, [`docker events`](https://docs.docker.com/reference/cli/docker/system/events/) los eventos del motor y [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) el estado de Compose. [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) muestra el contexto del daemon bajo systemd. Docker documenta por separado drivers de logging, rotación y dual logging; logs JSON ilimitados pueden llenar el host ([Docker – Configure logging drivers](https://docs.docker.com/engine/logging/configure/)).

Los health checks pueden detectar un proceso defectuoso y un orquestador puede reiniciarlo. Sin embargo, no reparan datos perdidos, configuraciones incorrectas ni una dependencia externa rota.

## Reiniciar no es una estrategia de recuperación

Las políticas de reinicio reaccionan a la finalización de procesos. No conocen datos dañados, colas bloqueadas ni credenciales incorrectas. Docker también indica que una política de reinicio solo se activa cuando un contenedor se ejecutó correctamente durante al menos diez segundos; las detenciones manuales la suprimen hasta el reinicio del daemon o el inicio manual ([Docker – Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)).

Los bucles de reinicio consumen CPU, generan logs y pueden sobrecargar servicios dependientes. Los orquestadores usan backoff, pero el administrador sigue necesitando el primer error, el código de salida, la causa de OOM/eviction, la última configuración y la línea temporal de eventos. Un CrashLoop es un estado sintomático, no una causa.

## Orquestación: pod, nodo y estado deseado

Kubernetes agrupa uno o varios contenedores en un **pod**. Los contenedores de un pod comparten el espacio de nombres de red y pueden compartir volúmenes; se programan conjuntamente. Los deployments gestionan ReplicaSets y actualizaciones escalonadas. En el nodo, kubelet implementa el PodSpec mediante CRI, CNI y plugins de almacenamiento ([Kubernetes – Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)).

| Nivel | Propietario | Incidencia típica |
|---|---|---|
| Plano de control | API, scheduler, controller | El estado deseado no se calcula o programa |
| Nodo/kubelet | Implementación local | Image pull, presión de disco/PID/memoria, error de runtime o CNI |
| Sandbox de pod | Red de pod compartida | Creación de sandbox/IP o pérdida de espacio de nombres |
| Contenedor | Imagen y proceso | Error de inicio, configuración, salida u OOM |
| Servicio/Ingress | Accesibilidad y enrutamiento | Sin endpoints listos, puerto o política incorrectos |
| Volumen persistente | Ruta de datos | Attach/mount, zona, permisos, snapshot o backend |

La fase `Running` significa que al menos un contenedor principal se está ejecutando o iniciando; no es un SLA de aplicación. Se evalúan conjuntamente el estado del contenedor, las conditions, los eventos, las probes y el estado del controller ([Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)).

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

[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/), [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) y [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) muestran la perspectiva de API, eventos y contenedores. [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) examina la vista CRI en el nodo. Un `kubectl exec` modifica y observa la instancia de runtime; no sustituye una imagen reproducible ni un runbook.

## Compose y sistemas declarativos de un solo host

Compose describe servicios, redes, volúmenes, secretos, configuraciones y dependencias en un modelo de aplicación. Es valioso para flujos de un solo host y desarrollo, pero no es ni un registro ni un orquestador de clústeres. La Compose Specification define el modelo independientemente de una CLI concreta ([Compose Specification](https://compose-spec.io/)).

Un repositorio Compose apto para producción contiene:

- digests de imágenes o resolución controlada de etiquetas y plataforma documentada;
- montajes explícitos, sistema de archivos raíz de solo lectura y rutas escribibles;
- semántica de recursos, reinicio, detención y salud;
- configuración separada y referencias de secretos;
- redes y puertos publicados;
- requisitos de logging y rotación;
- procedimientos de copia de seguridad/restauración y actualización fuera del archivo YAML.

`docker compose config` renderiza la configuración combinada y muestra la resolución de variables. La salida puede contener secretos y se trata en consecuencia ([Docker – `docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/)).

## Image pull, registro y caché

Un registro distribuye contenido mediante manifiestos y blobs. La autenticación, la autorización de repositorios, la resolución de etiquetas, los mirrors, la caché de proxy y los almacenes de contenido locales pueden fallar de forma independiente. Kubernetes `imagePullPolicy` decide cuándo kubelet contacta el registro; también `Always` utiliza capas presentes localmente si el digest resuelto ya existe ([Kubernetes – Images](https://kubernetes.io/docs/concepts/containers/images/)).

Un despliegue productivo registra el registro, repositorio, etiqueta, digest resuelto, plataforma, decisión de firma/provenance e inventario de nodos. La recolección de basura en el registro o nodo no debe eliminar digests aún necesarios para un rollback. La operación air-gapped necesita además un proceso para mirror, claves, revocación y metadatos.

## Seguridad de la cadena de suministro y del runtime

NIST distingue los riesgos en imágenes, registros, orquestadores, contenedores y sistemas operativos host. El resultado de un escáner es solo una señal: el inventario de paquetes puede estar incompleto, una CVE puede no ser alcanzable o, a la inversa, un error propio de la aplicación puede ser invisible. La política vincula procedencia, firma, vulnerabilidades conocidas, configuración y contexto de runtime ([NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final)).

Los Pod Security Standards de Kubernetes definen los perfiles Privileged, Baseline y Restricted. Restricted exige, entre otros, ejecución non-root, un perfil Seccomp y capabilities muy limitadas; los workloads concretos deben seguir probándose funcionalmente ([Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

Como mínimo se comprueba:

- registro de confianza y digest inmutable;
- provenance de compilación, firma y fuente de claves/identidad controlada;
- imagen base mínima y ningún secreto de compilación en capas;
- non-root, capability-drop, Seccomp/LSM, rootfs de solo lectura, dispositivos y montajes limitados;
- ningún socket de runtime ni espacio de nombres de host sin excepción explícita;
- Network Policies o firewall del host y egress controlado;
- límites de recursos y PID contra agotamiento local;
- ruta de parche, reconstrucción, despliegue y rollback.

## Contenedores de Windows

Los contenedores de Windows utilizan imágenes Windows y mecanismos del kernel de Windows. Microsoft distingue **Process Isolation**, en la que los contenedores comparten el kernel del host, y **Hyper-V Isolation**, en la que cada contenedor se ejecuta en una VM optimizada con su propio kernel. Ambos utilizan el mismo formato de imagen y las mismas herramientas de gestión, pero tienen límites de aislamiento y compatibilidad distintos ([Microsoft Learn – Windows and containers](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/), [Microsoft Learn – Isolation modes](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)).

Con Process Isolation, el sistema operativo del host y del contenedor deben ser compatibles. Hyper-V Isolation puede desacoplar determinadas diferencias de versión, pero aumenta los costes de recursos y arranque. Microsoft documenta las combinaciones admitidas de host e imagen; por eso «Windows Container» es incompleto sin indicar imagen base, build y aislamiento ([Microsoft Learn – Windows container version compatibility](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)).

Los nodos Linux y Windows no comparten el mismo kernel ni los mismos binarios de imagen. Los clústeres multi-OS necesitan etiquetas de programación, DaemonSets adecuados, plugins de red/almacenamiento y rutas de diagnóstico diferentes.

## Actualizaciones, despliegue y rollback

Una actualización de contenedor es un **cambio de imagen más una transición de estado**. Antes del despliegue se revisan las notas de versión, cambios de esquema, pasos de migración, versiones mínimas de servicios externos, nuevos puertos/scopes y compatibilidad hacia atrás. Un rollback de la imagen puede ser imposible tras una migración de datos no compatible hacia atrás.

El procedimiento controlado es:

1. registrar el digest objetivo, la firma, la provenance y la decisión del escaneo;
2. crear una copia de seguridad o un punto de recuperación probado de los datos persistentes;
3. revisar el diff de configuración y esquema;
4. desplegar un canary o réplicas escalonadas con probes y SLO reales;
5. observar la migración de datos y la capacidad de versiones mixtas;
6. verificar digest, instancias, eventos, errores, latencia y recursos;
7. activar el rollback solo dentro de la compatibilidad de datos demostrada.

Este proceso conecta [Releases](/kb/releases), [Migración](/kb/migration) y [Backup/DR](/kb/backup-dr). Los actualizadores automáticos de etiquetas sin gates funcionales simplemente trasladan el momento del cambio desde el proceso de cambio a un bot.

## Copia de seguridad y recuperación ante desastres

Un servicio de contenedores completo consta de más que volúmenes:

- definiciones de despliegue, políticas y objetos de red;
- digests de imágenes registrados o un mirror de registro restaurable;
- configuración no sensible y fuentes de secretos/claves;
- datos persistentes con un procedimiento consistente para la aplicación;
- bases de datos externas, colas, object stores y dependencias de DNS e identidad;
- versiones de esquema, trabajos abiertos y estado de reconciliación;
- runbooks para pérdida de nodo, clúster, registro y ubicación.

Un archivo tar de la ruta del volumen puede ser inconsistente si la base de datos está en ejecución. Los snapshots de almacenamiento requieren semántica de freeze/quiesce o específica de base de datos. Una prueba de restauración reconstruye el servicio, la red y las identidades en un entorno de destino limpio, inicia con el digest protegido y comprueba los datos funcionales y las colas abiertas.

Ante incidencias se verifica desde el artefacto hacia el proceso y después hacia fuera: imagen, parámetros de inicio, permisos, montajes, resolución de nombres, ruta de red y servicios externos.

## Diagnóstico por capas de dependencia

Una incidencia de contenedor se acota de fuera hacia dentro:

1. **Estado deseado:** ¿Qué definición y qué digest deberían ejecutarse?
2. **Placement:** ¿En qué host/nodo, con qué plataforma y capacidad?
3. **Imagen:** ¿Pull, autenticación, manifiesto, plataforma, firma y contenido local?
4. **Runtime:** ¿Sandbox, estado del contenedor, código de salida, OOM, reinicio y eventos?
5. **Proceso:** ¿PID 1, señales, usuario, capabilities y archivos abiertos?
6. **Almacenamiento:** ¿Montaje esperado, backend, permisos, capacidad y E/S?
7. **Red:** ¿Espacio de nombres, DNS, ruta, política, listener, servicio y TLS?
8. **Aplicación:** ¿Salud, logs, cola, esquema, credencial y dependencia externa?

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

[`docker version`](https://docs.docker.com/reference/cli/docker/version/) distingue la versión de cliente y servidor, [`docker info`](https://docs.docker.com/reference/cli/docker/system/info/) muestra el contexto de motor, runtime, almacenamiento y seguridad, [`docker context show`](https://docs.docker.com/reference/cli/docker/context/show/) el objetivo realmente administrado y [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) el consumo de contenido local. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html) acredita el kernel y la plataforma en el ejemplo Unix.

## Historia técnica

El aislamiento de procesos es anterior a las imágenes modernas. El `chroot` de Unix cambiaba la raíz del sistema de archivos de un proceso, pero nunca se concibió como un límite de seguridad completo. FreeBSD Jails amplió el modelo a finales de la década de 1990, respectivamente con FreeBSD 4.0, con vistas más aisladas del host y la red; Solaris Zones combinó aislamiento de aplicaciones y gestión de recursos en el sistema operativo ([FreeBSD Handbook – Jails](https://docs.freebsd.org/en/books/handbook/jails/), [Oracle Solaris Zones Introduction](https://docs.oracle.com/cd/E37838_01/html/E61039/zonesintro.html)).

Linux introdujo gradualmente espacios de nombres y Control Groups. LXC combinó estos mecanismos del kernel en contenedores de sistema. Docker popularizó desde 2013 las capas de imagen, la distribución mediante registro, los Dockerfiles y una interfaz de desarrollador coherente; inicialmente utilizaba LXC y posteriormente migró a una biblioteca de runtime propia ([Linux Containers – LXC Introduction](https://linuxcontainers.org/lxc/introduction/), [Docker – What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)).

En 2015, fabricantes y proveedores de plataformas fundaron la Open Container Initiative para estandarizar abiertamente los formatos de runtime e imagen. `runc` se convirtió en la base del runtime OCI; `containerd` y CRI-O establecieron runtimes de alto nivel. Kubernetes abstrajo los runtimes de nodo mediante CRI, las redes mediante CNI y el almacenamiento mediante CSI. Por ello, hoy «contenedor» no designa una única pila de productos, sino una cadena de especificaciones e implementaciones interoperables ([Open Container Initiative – Overview](https://opencontainers.org/about/overview/), [Kubernetes – Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)).

## Lista de comprobación para administradores

El contenedor es solo una parte del servicio. Por ello, para la aprobación operativa se revisan el artefacto, el runtime, el host, la red, el almacenamiento y la recuperación como una cadena coherente.

| Pregunta | Evidencia operativa |
|---|---|
| ¿Qué artefacto se ejecuta? | Registro, repositorio, etiqueta, digest, plataforma, digest de configuración, firma/provenance |
| ¿Qué cadena de runtime se aplica? | Motor/orquestador, CRI, runtime de alto nivel, runtime OCI, kernel del host |
| ¿Qué aislamiento está activo? | Espacios de nombres, modo de aislamiento de Windows, mapeo de UID, capabilities, Seccomp, LSM |
| ¿Dónde reside el estado? | Writable Layer, volumen, Bind Mount, tmpfs y servicios externos por clase de datos |
| ¿Qué es accesible? | Dirección de escucha, IP de contenedor/pod, servicio, puerto publicado, Ingress y política de egress |
| ¿Quién posee la identidad? | UID/SID de proceso, Service Account, credencial de registro, token de workload y fuente de claves |
| ¿Qué límites se aplican? | CPU, memoria, PID, E/S, disco, rotación de logs, cuota y eviction de nodo |
| ¿Qué significa saludable? | Proceso, startup, readiness, liveness, prueba funcional y dependencias externas por separado |
| ¿Cómo se modifica? | Digest aprobado, diff de esquema/configuración, canary, versión mixta, ventana de rollback |
| ¿Cómo se restaura? | Definiciones, registro/imágenes, secretos/claves, datos, colas y prueba de restauración limpia |

## Fuentes

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
