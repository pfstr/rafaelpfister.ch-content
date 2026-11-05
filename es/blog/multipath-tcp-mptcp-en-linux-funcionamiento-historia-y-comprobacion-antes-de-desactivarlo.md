---
title: "Multipath TCP (MPTCP) en Linux: funcionamiento, historia y comprobación antes de desactivarlo"
navTitle: "MPTCP en Linux"
description: "Multipath TCP distribuye una conexión TCP entre varias rutas de red. Desde Go 1.24, muchos servicios de Go escuchan mediante MPTCP en Linux sin que nadie lo haya configurado. Con CVE-2026-98432, esto se ha vuelto relevante. El artículo explica el protocolo, su historia y cómo comprobar si puede desactivar MPTCP."
date: "2026-11-02"
kategorie: "Kernel de Linux"
timeToRead: "10 min de lectura"
themen:
  - linux-kernel
  - sicherheitsluecken
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-en-linux-funcionamiento-historia-y-comprobacion-antes-de-desactivarlo"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/es/blog/multipath-tcp-mptcp-en-linux-funcionamiento-historia-y-comprobacion-antes-de-desactivarlo
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:32:58.368Z
translationReview: automatic
---

Al comprobar mi servidor doméstico frente a CVE-2026-98432, descubrí que varios servicios escuchaban mediante Multipath TCP, entre ellos Prometheus-node-exporter, Pi-hole-Exporter y `tailscaled`, todos escritos en Go. Nunca había configurado esto. El exploit de root descrito públicamente hasta ahora para esta vulnerabilidad requiere precisamente un listener MPTCP de este tipo.

Conclusión breve: MPTCP forma parte del kernel oficial de Linux desde el kernel 5.6 y está permitido de forma predeterminada. Desde Go 1.24, los programas de Go abren automáticamente sus listeners como sockets MPTCP. Puede determinar si es posible desactivar MPTCP con dos comandos y algunos contadores del kernel. En la mayoría de los servidores ningún cliente utiliza el protocolo, y Go vuelve automáticamente a TCP normal tras desactivarlo.

## Qué es Multipath TCP

Una conexión TCP normal está vinculada a exactamente un par de direcciones: IP y puerto de origen, IP y puerto de destino. Si un smartphone cambia de Wi-Fi a red móvil, cambia la IP de origen y la conexión se interrumpe. Multipath TCP amplía TCP para que una conexión pueda utilizar varias rutas simultánea o sucesivamente, por ejemplo Wi-Fi y red móvil.

Técnicamente, una conexión MPTCP consta de varios `Subflows`. Cada subflow es para la red una conexión TCP normal a través de una ruta. La aplicación no percibe nada de ello: trabaja con un único socket y el kernel distribuye los datos entre los subflows. Para que los datos lleguen en el orden correcto, MPTCP mantiene su propio número de secuencia de 64 bits a nivel de conexión (Data Sequence Number), además de los números de secuencia de cada subflow.

MPTCP se negocia en el handshake TCP normal. SYN, SYN/ACK y el primer ACK incluyen la opción TCP `MP_CAPABLE` (Option Kind 30), con la que ambas partes declaran que hablan MPTCP e intercambian claves. Posteriormente, otros subflows se unen a la conexión con la opción `MP_JOIN`. Si falta `MP_CAPABLE` en la respuesta porque el otro extremo no conoce MPTCP o porque una middlebox elimina la opción, la conexión continúa como TCP normal. Este fallback forma parte del estándar y es el motivo por el que los servidores compatibles con MPTCP funcionan con todos los clientes TCP.

En Linux, MPTCP es opt-in por socket: una aplicación debe crear explícitamente el socket con el protocolo `IPPROTO_MPTCP` (valor 262). Un programa que no lo haga seguirá utilizando TCP normal, aunque MPTCP esté activo en el kernel.

## La historia

MPTCP surgió en universidades y entró en funcionamiento a gran escala a través de Apple. La implementación de Linux permaneció más de una década fuera del kernel oficial antes de adoptarse en una nueva forma.

| Momento | Evento |
|---|---|
| desde aproximadamente 2009 | Investigadores de la Université catholique de Louvain (UCLouvain) desarrollan la primera implementación de Linux como una rama propia del kernel (multipath-tcp.org). |
| marzo de 2011 | RFC 6182 describe la arquitectura de Multipath TCP (Informational). |
| enero de 2013 | RFC 6824 especifica MPTCP versión 0 como protocolo experimental. |
| septiembre de 2013 | Se lanza iOS 7 con MPTCP, el primer uso comercial a gran escala. Apple utiliza MPTCP, entre otras cosas, para Siri. |
| junio de 2015 | KT (Korea Telecom) lanza un servicio comercial que combina Wi-Fi y LTE mediante MPTCP y permite hasta 1 Gbit/s en smartphones; presentado en la IETF 93. |
| 2017 | iOS 11 abre MPTCP a desarrolladores de aplicaciones mediante `URLSession` (`multipathServiceType`). |
| marzo de 2020 | RFC 8684 define MPTCP versión 1 como Proposed Standard y sustituye RFC 6824. |
| 29 de marzo de 2020 | Se lanza Linux 5.6 con un primer soporte de MPTCP en el kernel oficial, reescrito y solo para la versión 1. Los subflows adicionales llegaron en versiones posteriores. |
| 2020 | 3GPP especifica ATSSS (Access Traffic Steering, Switching, Splitting) para 5G en Release 16; allí MPTCP distribuye el tráfico entre red móvil y Wi-Fi. |
| agosto de 2023 | Go 1.21 introduce `SetMultipathTCP` para dialers y listeners, desactivado de forma predeterminada. |
| febrero de 2025 | Go 1.24 activa MPTCP de forma predeterminada para listeners (`ListenConfig`), mientras que los dialers siguen usando TCP. |
| octubre de 2026 | CVE-2026-98432: use-after-free en el código IPv4-TCP; el exploit de root mostrado requiere un listener MPTCP. |

La rama out-of-tree de UCLouvain admitía las versiones 0 y 1. El kernel oficial solo conoce la versión 1; actualmente el proyecto recomienda exclusivamente esta implementación.

## Por qué MPTCP está activo en muchos servidores

Coinciden dos valores predeterminados. El kernel permite sockets MPTCP de forma predeterminada: el ajuste sysctl `net.mptcp.enabled` está establecido en `1`. Además, desde la versión 1.24 los programas de Go crean cada listener como socket MPTCP siempre que el kernel lo permita. Por tanto, servicios de Go como exportadores de Prometheus o Tailscale escuchan mediante MPTCP tan pronto como se compilan con Go 1.24 o posterior.

Para los clientes normales esto no cambia nada: no envían `MP_CAPABLE` y la conexión funciona como TCP. Sin embargo, la superficie de ataque adicional surge en el servidor, ya que cualquier cliente que incluya `MP_CAPABLE` alcanza el código MPTCP del kernel.

## CVE-2026-98432

CVE-2026-98432 es un use-after-free al establecer conexiones IPv4-TCP con opciones IP (CVSS 3.1: 9.8). Según la entrada CVE, Linux está afectado desde la versión 4.4. El error en sí afecta a todos los listeners TCP, no solo a MPTCP. Sin embargo, hasta ahora solo se ha mostrado una cadena de explotación hasta privilegios de root a través de un listener MPTCP; según lilting.ch, Anthropic la demostró el 27 de octubre de 2026 en un entorno de laboratorio. No está claro si también pueden alcanzarse privilegios de root sin MPTCP.

Upstream, el error está corregido en los kernels Stable 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 y 7.2.10, así como en Mainline 7.3 (commit «ipv4: free inet_opt and ireq_opt after an RCU grace period"). El 30 de octubre de 2026, Ubuntu seguía indicando las versiones LTS 22.04, 24.04 y 26.04 como «Vulnerable, work in progress"; para 26.04, la corrección está prevista en `linux` 7.0.0-43.43. Las distribuciones retroportan correcciones; por ello, lo decisivo es la versión del paquete del aviso de la distribución, no la versión principal del kernel.

Desactivar MPTCP elimina el requisito del exploit mostrado. La vulnerabilidad en el código IPv4-TCP sigue existiendo hasta que se instale y arranque el kernel actualizado.

## Comprobar si MPTCP está activo y se utiliza

### 1. ¿Permite el kernel MPTCP?

```bash
sysctl net.mptcp.enabled
```

`1` significa que las aplicaciones pueden crear sockets MPTCP. Si `sysctl` informa de que la clave es desconocida, MPTCP no está compilado en el kernel y no está disponible en este host.

### 2. ¿Qué servicios escuchan mediante MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-M` | mostrar solo sockets MPTCP |
| `-l` | mostrar solo sockets en escucha (listeners) |
| `-n` | direcciones y puertos numéricos, sin resolución de nombres |
| `-p` | mostrar el proceso asociado al socket; requiere privilegios de root para procesos de otros usuarios |

</details>

Cada línea es un listener MPTCP. El mismo puerto aparece adicionalmente como listener TCP con `ss -tln`; esto es normal, porque el kernel mantiene internamente un socket TCP por cada listener MPTCP. Los nombres de procesos muestran qué programas están afectados. Por regla general, son programas de Go.

### 3. ¿Ha utilizado algún cliente MPTCP alguna vez?

El kernel cuenta los handshakes MPTCP desde el último arranque. Los contadores relevantes son:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-a` | valores absolutos desde el arranque del sistema en lugar de la diferencia respecto a la última llamada |
| `-s` | no actualizar el archivo de historial de `nstat` |
| `-z` | mostrar también contadores con valor 0 |

</details>

| Contador | Significado |
|---|---|
| `MPTcpExtMPCapableSYNRX` | SYN entrantes con `MP_CAPABLE` en un listener MPTCP |
| `MPTcpExtMPCapableACKRX` | conexiones MPTCP entrantes cuyo handshake se completó |
| `MPTcpExtMPCapableSYNTX` | SYN salientes con `MP_CAPABLE`, es decir, conexiones que este host quiso establecer mediante MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | conexiones salientes en las que el otro extremo confirmó MPTCP |
| `MPTcpExtMPCapableFallbackACK` | conexiones que volvieron a TCP en el lado del servidor durante el handshake |
| `MPTcpExtMPCapableFallbackSYNACK` | conexiones que volvieron a TCP en el lado del cliente |
| `MPTcpExtMPCurrEstab` | conexiones MPTCP actualmente existentes |

Si `MPCapableSYNRX` y `MPCapableACKRX` siguen en 0 tras un tiempo de funcionamiento representativo, ningún cliente ha utilizado MPTCP. Si `MPCapableSYNTX` es mayor que 0, una aplicación del propio host establece conexiones MPTCP salientes; debe identificar esa aplicación antes de desactivarlo. `ss -Mani` muestra las conexiones MPTCP existentes con detalles de los subflows.

Los contadores son válidos desde el último reinicio. Por ello, un servidor que solo se inició hace una hora no proporciona una conclusión fiable. Además, el kernel cuenta por espacio de nombres de red; encontrará más información en la siguiente sección.

### 4. Comprobar los contenedores por separado

`net.mptcp.enabled` se aplica por espacio de nombres de red. Cada contenedor con su propia red (bridge de Docker, Podman, pods de Kubernetes) tiene su propio valor, y un espacio de nombres nuevo comienza con el valor predeterminado `1`, independientemente de la configuración del host. En mi servidor doméstico, tras desactivarlo en el host, todos los contenedores bridge seguían en `1`; solo los contenedores con `network_mode: host` adoptaron el `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `docker ps -q` | IDs de todos los contenedores en ejecución |
| `docker inspect -f '{{.State.Pid}}'` | PID del proceso principal del contenedor, mediante el cual `nsenter` encuentra el espacio de nombres |
| `nsenter -t <pid>` | entrar en los espacios de nombres del proceso indicado |
| `-n` | entrar solo en el espacio de nombres de red; el sistema de archivos y las herramientas siguen siendo los del host |
| `sysctl -n` | mostrar solo el valor, sin el nombre de la clave |

</details>

`nsenter` tiene la ventaja de que el contenedor no necesita contener `sysctl` ni `cat`; muchas imágenes (distroless, scratch) no incluyen una shell. Del mismo modo se pueden consultar `ss -Mln` y los contadores de `nstat` en el espacio de nombres de un contenedor.

## Cuándo puede desactivar MPTCP

Desactivarlo no plantea problemas si los contadores del paso 3 no muestran uso de MPTCP en el host ni en los contenedores. Los programas de Go que escuchan no pierden nada por ello: Go intenta crear el socket MPTCP, recibe un error del kernel y crea silenciosamente un socket TCP. Otros programas que solicitan explícitamente `IPPROTO_MPTCP` reciben el error `ENOPROTOOPT`. Que después vuelvan a TCP o terminen depende del software; pruebe dichos servicios tras el cambio.

Conviene tener precaución en estos casos:

- **Acceso híbrido y agregación de enlaces:** routers o proxies que combinan DSL, fibra y red móvil mediante MPTCP (como OpenMPTCProuter o gateways de acceso híbrido de proveedores) dependen de MPTCP.
- **Servidores para aplicaciones móviles:** una aplicación propia de iOS con `multipathServiceType` solo se beneficia al cambiar de red si el servidor habla MPTCP. Los contadores `MPCapableACKRX` muestran si existen esos clientes.
- **MPTCP saliente:** los programas iniciados con `mptcpize` o que se ejecutan en Go con `GODEBUG=multipathtcp=1` o `3` establecen MPTCP saliente. Se reconoce por `MPCapableSYNTX` mayor que 0.

## Desactivar MPTCP

Hay tres opciones, según el alcance deseado.

### Para todo el host

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | archivo de configuración que se lee al arrancar; el ajuste persiste tras reinicios |
| `sysctl --system` | recargar todos los archivos bajo `/etc/sysctl.d/` y las demás rutas predeterminadas, y aplicarlos inmediatamente |

</details>

El ajuste solo evita la creación de nuevos sockets MPTCP. Los listeners ya abiertos siguen siendo sockets MPTCP hasta que se reinicie el servicio. Lo más sencillo es reiniciar el host; después, `ss -Mln` debe devolver una lista vacía.

### Para contenedores

En contenedores con red propia, establezca el valor por contenedor. Docker permite sysctls del ámbito `net.*` siempre que el contenedor no utilice la red del host:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

En la llamada directa equivale a `docker run --sysctl net.mptcp.enabled=0 …`. El cambio se aplica al volver a crear el contenedor (`docker compose up -d`).

### Solo para programas de Go concretos

En lugar de desactivar MPTCP en el kernel, puede restablecer el comportamiento predeterminado de Go por servicio. La variable de entorno `GODEBUG=multipathtcp=0` desactiva MPTCP para listeners y dialers. En un servicio systemd, puede hacerse mediante un drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Si el servicio ya establece una variable `GODEBUG`, las entradas se combinan en una variable separadas por comas. Esta posibilidad solo funciona con programas de Go y solo si el código no llama por sí mismo a `SetMultipathTCP`; una llamada explícita en el programa tiene prioridad sobre `GODEBUG`. Para proyectos propios de Go, el mismo ajuste está disponible como `godebug multipathtcp=0` en `go.mod` o como `//go:debug multipathtcp=0` en el paquete principal.

## Comprobación tras el cambio

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Se esperan `0` y ninguna línea bajo la cabecera de tabla de `ss`. Después, los servicios siguen escuchando en los mismos puertos, visibles con `ss -tln`. Puede revertir el cambio eliminando el archivo bajo `/etc/sysctl.d/` y reiniciando el host.

## Fuentes

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): especificación de MPTCP versión 1 (marzo de 2020) con subflows, `MP_CAPABLE`, Data Sequence Number y fallback.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP versión 0, experimental (enero de 2013), sustituida por RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): documento de arquitectura de marzo de 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): opt-in mediante `IPPROTO_MPTCP`, códigos de error al crear sockets, Path Manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): significado y valor predeterminado de `net.mptcp.enabled` y los demás sysctls de MPTCP.

6.  [mptcp.dev](https://www.mptcp.dev/): sitio web del proyecto upstream con FAQ, guía de depuración (`ss -Mani`, `nstat`) e indicaciones sobre macOS e iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): resumen de la versión con el primer soporte de MPTCP en el kernel oficial.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): introducción del soporte de MPTCP en el paquete `net`, desactivado de forma predeterminada, con fallback silencioso a TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP activado de forma predeterminada para `ListenConfig` en Linux.

10.  [Go: ajustes de GODEBUG](https://go.dev/doc/godebug): valores de 0 a 3 de `multipathtcp`, así como `godebug` en `go.mod` y `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): visión general de implementaciones y usos (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): estado de los kernels de Ubuntu por versión y versiones de corrección previstas.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): evaluación de la explotabilidad, función de MPTCP y versiones de corrección de los kernels Stable.
