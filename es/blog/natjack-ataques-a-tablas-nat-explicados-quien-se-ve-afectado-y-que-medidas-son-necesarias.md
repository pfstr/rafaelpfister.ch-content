---
title: "NatJack: ataques a tablas NAT explicados, quién se ve afectado y qué medidas son necesarias"
navTitle: "NatJack explicado"
description: "NatJack es una nueva clase de ataque contra dispositivos NAT: un atacante detrás del mismo router, firewall o puente Docker puede secuestrar conexiones TCP ajenas, interceptar respuestas DNS o dejar inoperativo el NAT. Cómo funciona el ataque, qué productos se ven afectados, cuándo son necesarias medidas y cuáles son eficaces."
date: "2026-10-28"
kategorie: "Vulnerabilidades de seguridad"
timeToRead: "10 min de lectura"
themen:
  - sicherheitsluecken
produkte:
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "dns"
  - "container"
  - "haertung"
hauptthema: "sicherheitsluecken"
slug: "natjack-ataques-a-tablas-nat-explicados-quien-se-ve-afectado-y-que-medidas-son-necesarias"
translationId: "article-223ba44d60e8cbc8"
translationOf: natjack-nat-tabellen-angriff
url: https://rafaelpfister.ch/es/blog/natjack-ataques-a-tablas-nat-explicados-quien-se-ve-afectado-y-que-medidas-son-necesarias
translationSourceHash: c29a779430bbe591ef42a7db22926ad80fea50613dbc60458b9e88e40edadc02
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T21:31:40.575Z
translationReview: automatic
---

# NatJack: ataques a tablas NAT explicados, quién se ve afectado y qué medidas son necesarias

NatJack es una clase de ataque contra dispositivos que realizan traducción de direcciones de red (NAT): routers, firewalls, redes de hipervisores, puentes de contenedores y puertas de enlace NAT en la nube. Un atacante que se encuentra detrás del mismo dispositivo NAT que su víctima manipula su tabla NAT de modo que las respuestas procedentes de Internet ya no llegan a la víctima, sino a él. Así se pueden secuestrar conexiones TCP existentes, interceptar y falsificar respuestas DNS o bloquear el dispositivo NAT para todos los demás usuarios. La investigación fue publicada por Malcolm Stagg (SODIUM-24, LLC) el 6 de agosto de 2026 en Black Hat USA, coordinada a través de CERT/CC (VU#268870) con 13 fabricantes. En 23 productos probados se notificaron 95 hallazgos individuales; según el white paper, todos los dispositivos probados con NAT eran vulnerables a al menos una parte de los ataques.

Conclusión breve: no es necesario actuar en una red doméstica con dispositivos propios. Las medidas son necesarias allí donde usuarios y cargas de trabajo externos o no confiables utilizan el mismo dispositivo NAT que sus sistemas de producción: Wi-Fi para invitados en el firewall corporativo, hosts Docker y Kubernetes con imágenes ajenas, máquinas virtuales de prueba en la red NAT de un host Hyper-V, entornos de hosting y coworking. La medida más eficaz es el cifrado (TLS también internamente, DNS protegido); la segunda más eficaz es separar las zonas de confianza en dispositivos NAT propios.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Asistencia para la evaluación</p>
<p>Si desea comprobar si su entorno de firewall, contenedores o hipervisor se ve afectado, utilice el <a href="https://adeptio.ch/en">formulario de contacto en adeptio.ch</a>.</p>
</div>

## Qué hace una tabla NAT

Detrás de un dispositivo NAT, los dispositivos tienen direcciones privadas de RFC 1918, por ejemplo `10.0.10.21`. Cuando un dispositivo establece una conexión hacia el exterior, el dispositivo NAT sustituye la dirección y el puerto de origen por su dirección externa y un puerto externo, y guarda la asignación en la tabla NAT. Cuando vuelve la respuesta del servidor, el dispositivo consulta la tabla para saber a qué dispositivo interno pertenece. En Linux, Netfilter se encarga de ello con su seguimiento de conexiones `conntrack`; la mayoría de los routers comerciales, OpenWRT, Docker y Kubernetes utilizan este código.

Originalmente, NAT era una solución transitoria para la escasez de direcciones IPv4 y no un límite de seguridad. Sin embargo, en la práctica se ha convertido en uno: en la capa 2, las VLAN, el aislamiento de puertos, Dynamic ARP Inspection y el aislamiento de vSwitch separan correctamente los dispositivos entre sí, y el ARP spoofing apenas es posible en redes bien mantenidas. Sin embargo, en la capa 3/4 las redes separadas suelen converger de nuevo en el mismo dispositivo NAT. Precisamente ahí es donde actúa NatJack.

![Dispositivo NAT compartido con dos VLAN separadas: la entrada NAT de la víctima se sustituye por una entrada del atacante y las respuestas del servidor de destino pasan a dirigirse al atacante](../images/natjack-nat-tabellen-angriff/geteiltes-nat.svg)

## Cómo funciona el ataque

Todas las variantes de NatJack siguen el mismo patrón: el atacante consigue que el dispositivo NAT elimine la entrada de tabla de la víctima e inmediatamente crea una entrada propia con los mismos parámetros externos. A continuación, el dispositivo NAT redirige la siguiente respuesta del servidor al atacante. Ni el servidor ni la víctima reciben un mensaje de error.

Esto es posible por comportamientos diseñados para una red cooperativa y que allí tienen sentido:

- **Modo laxo:** De forma predeterminada, Netfilter también incorpora a la tabla conexiones TCP cuyo handshake no ha visto. Esto pretende conservar conexiones tras el reinicio de un router o un cambio de ruta, pero también permite crear entradas sin un establecimiento de conexión válido.
- **Comprobación permisiva de paquetes de reset:** El dispositivo NAT comprueba los números de secuencia con menos rigor que los propios extremos para no descartar paquetes válidos. Los paquetes que un extremo ignoraría todavía pueden ser considerados por el dispositivo NAT como fin de conexión.
- **Preservación de puertos y mapeo independiente del extremo:** Netfilter utiliza, cuando es posible, el puerto interno como puerto externo y reutiliza para todos los destinos una asignación ya concedida. El comportamiento ante conflictos de puertos permite deducir qué puertos están utilizando otros dispositivos en ese momento.
- **Tabla finita:** Si la tabla está llena, Netfilter descarta aleatoriamente entradas aún no confirmadas, como consultas DNS abiertas. Las conexiones confirmadas permanecen hasta cinco días.

Para la mayoría de las variantes, el atacante debe poder enviar paquetes con una dirección de origen falsificada. Si atacante y víctima se encuentran en la misma subred, le basta un dispositivo o contenedor con permisos de raw socket (`NET_RAW`). La variante upstream también funciona a través de límites de subred y VLAN; para ello, el atacante necesita además un servidor propio en Internet que pueda enviar paquetes con una dirección de origen falsificada. El white paper describe las secuencias de paquetes y el código de prueba de concepto; el Technical Report anunciado con el marco de pruebas aún no se ha publicado.

## Las cinco vulnerabilidades de un vistazo

El white paper describe cinco vulnerabilidades. Las valoraciones son valores típicos; pueden variar según el producto.

| Vulnerabilidad | Impacto | CVSS 3.1 (típico) |
|---|---|---|
| Denegación de servicio | El atacante llena la tabla NAT con entradas confirmadas. Ningún otro dispositivo puede establecer nuevas conexiones, en Netfilter hasta cinco días o hasta reiniciar. | 7.4 (High) |
| Divulgación de puerto e IP | El atacante detecta que otro dispositivo está conectado con un servidor concreto y determina su puerto y dirección interna. Sirve principalmente como paso previo para los demás ataques. | 4.7 (Medium) |
| Secuestro de DNS mediante UDP | Con cierta probabilidad, el atacante intercepta la respuesta DNS para otro dispositivo y envía una respuesta falsificada, por ejemplo con la dirección de un servidor propio. | 10.0 (Critical) |
| Secuestro de TCP (downstream) | El atacante toma control de una conexión TCP existente de un dispositivo en la misma subred, lee las respuestas del servidor y envía datos en nombre de la víctima. | 9.6 (Critical) |
| Secuestro de TCP (upstream) | Igual que arriba, pero a través de límites de subred y VLAN con ayuda de un servidor externo del atacante. | 10.0 (Critical) |

Se ven afectados todos los protocolos sin cifrado sólido: HTTP, DNS mediante UDP, protocolos de administración basados en Telnet y protocolos de correo como SMTP e IMAP cuando funcionan sin TLS. En una conexión TLS, el atacante puede interrumpir la conexión, pero no obtiene datos legibles ni puede inyectar nada. El white paper también señala que los paquetes de ataque pueden enviarse con TTL reducido, de modo que lleguen al dispositivo NAT pero no continúen hacia la red. Esto dificulta su detección.

En comparación con el ARP spoofing, NatJack tiene un efecto similar, pero elude los mecanismos de protección establecidos en la capa 2. Trabajos anteriores de Yang et al. (CVE-2023-30305 a CVE-2023-30314) mostraron secuestro de TCP mediante tablas NAT en routers Wi-Fi con opciones no estándar inseguras; NatJack funciona con la configuración predeterminada.

## Quién se ve afectado

Según el white paper, prácticamente todos los routers basados en Linux son vulnerables, al igual que sistemas con una base de código completamente distinta. El siguiente resumen recoge el estado de la matriz de parches en natjack.io (última actualización: 19 de agosto de 2026).

| Área | Productos | Estado |
|---|---|---|
| Linux / Netfilter | Kernel Linux, OpenWRT, DD-WRT, casi todos los routers basados en Linux | Corrección parcial con CVE-2026-63913 a partir de Linux 7.1; aumenta la dificultad, sin protección completa. OpenWRT y DD-WRT necesitan los cambios de Netfilter. |
| BSD | FreeBSD pf, OpenBSD pf | FreeBSD a partir de 15.0.0 queda muy mitigado mediante un cambio procedente de OpenBSD; las versiones anteriores están afectadas. OpenBSD está considerablemente mejor protegido desde 2016. |
| Windows | Windows NAT (Hyper-V vSwitch con NAT) | CVE-2026-56181 (julio de 2026) y CVE-2026-56179 (agosto de 2026), ambos clasificados como moderados. La mitigación de CVE-2026-56179 está desactivada de forma predeterminada y se activa mediante el registro. Windows Internet Connection Sharing está en revisión. |
| Dispositivos de red | Cisco Firepower y Meraki, NETGEAR Nighthawk, Google Nest WiFi Pro, SpaceX Starlink, Amazon eero | Según el fabricante, protección parcial, recomendaciones de configuración o ninguna actualización. eero ha publicado un comunicado propio. |
| Hotspots | Hotspot personal de iOS, Compartir Internet de macOS, hotspot de Android | afectados |
| Virtualización y contenedores | Hyper-V, KVM/QEMU, Apple Virtualization Framework, Docker (Bridge), FreeBSD Jails, GKE, MicroK8s con Calico o Flannel, Azure AKS | afectados; protección principalmente mediante configuración (véase más abajo) |
| Nube | AWS NAT Gateway y NLB, Google Cloud NAT, Cloud Run, Azure Firewall, Azure NAT Gateway, Azure Functions | AWS ha reforzado el servicio en el servidor; los clientes no deben hacer nada. AWS Lambda está protegido por la ausencia de `NET_RAW`. Para los demás servicios, consulte el estado en natjack.io. |

## ¿Son necesarias medidas?

Lo decisivo es quién se encuentra detrás del mismo dispositivo NAT; que su modelo de router aparezca en la lista es secundario. Cada variante presupone que el atacante ya controla un dispositivo, una VM o un contenedor detrás del NAT. Cuando solo se trata de dispositivos propios y confiables, un atacante con ese acceso tendría en cualquier caso formas más sencillas. Por ello, el white paper indica que la mayoría de los usuarios domésticos no se ven seriamente afectados, mientras que las consecuencias para empresas y proveedores de servicios son significativamente más graves.

![Tres niveles: sin necesidad urgente de actuar con dispositivos exclusivamente propios, revisar y reforzar con redes de invitados separadas en el mismo firewall, medidas necesarias con usuarios o cargas de trabajo ajenos en el mismo NAT](../images/natjack-nat-tabellen-angriff/massnahmen-noetig.svg)

En las pymes suizas, el nivel intermedio es el caso más frecuente: la Wi-Fi para invitados está separada mediante VLAN, pero sale a Internet a través del mismo firewall que la red corporativa. La separación por VLAN evita el ARP spoofing y el acceso directo, pero ambas redes siguen compartiendo la tabla NAT. Para la variante downstream, el atacante necesitaría estar en la misma subred que su víctima; la variante upstream funciona a través del límite de VLAN, pero es más compleja y requiere un servidor externo. Las medidas son claramente necesarias donde terceros se encuentran en la misma subred (Wi-Fi de hoteles, consultas o escuelas sin aislamiento de clientes) y en hosts de contenedores y Kubernetes donde se ejecutan cargas de trabajo de distintos niveles de confianza. Un paquete npm o PyPI comprometido en un contenedor basta allí como punto de partida.

Para los servicios de correo conviene examinar específicamente las conexiones internas sin TLS: escáneres, aplicaciones especializadas y sistemas de monitorización que entregan sin cifrar a un relay interno mediante el puerto 25, así como IMAP sin TLS. El artículo [Envío de correo mediante un relay: comprobar TLS y autenticación](/blog/mail-relay-tls-authentisierung-pruefen) describe cómo comprobar TLS y la autenticación en el relay.

## Medidas

Las recomendaciones del white paper pueden ordenarse en cinco capas. Se diferencian en las vulnerabilidades contra las que ayudan.

![Cinco capas de protección: cifrado, separación de zonas de confianza, anti-spoofing, refuerzo de NAT y supervisión, cada una con su efecto sobre el secuestro, la divulgación de puertos y la denegación de servicio](../images/natjack-nat-tabellen-angriff/schutzschichten.svg)

### Cifrado

TLS en todas las conexiones, también entre sistemas internos, hace inútiles las conexiones TCP secuestradas: el atacante puede interrumpirlas, pero ni escuchar ni inyectar datos. Para DNS, el white paper recomienda DNSSEC contra respuestas falsificadas, así como DNS over HTTPS o DNS over TLS contra la interceptación. El cifrado no ayuda contra la divulgación de puertos ni la denegación de servicio.

### Separación de las zonas de confianza

Opere dispositivos y cargas de trabajo no confiables en subredes y VLAN propias, con ACL L3/L4 que impidan cualquier acceso a hosts confiables. Es más eficaz contar con un dispositivo NAT propio o una dirección externa propia por zona de confianza, pues sin una tabla NAT compartida falta la base para todas las variantes. En la nube, esto significa: no operar cargas de trabajo confiables y ajenas detrás del mismo NAT Gateway y utilizar direcciones IP dedicadas en lugar de compartidas para servicios serverless. En Kubernetes, los pods no confiables se ejecutan en nodos propios.

### Anti-spoofing

Active IP Source Guard o una función comparable en el switch o punto de acceso, y uRPF (Reverse Path Filtering) en el router, para que los clientes solo puedan enviar con su propia dirección. En los contenedores, retirar la capability `NET_RAW` evita que un proceso comprometido genere paquetes falsificados. Docker asigna esta capability de forma predeterminada.

```bash
docker run --cap-drop=NET_RAW --user 1000:1000 example/app:1.4
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `run` | Inicia un nuevo contenedor a partir de una imagen |
| `--cap-drop=NET_RAW` | Retira al contenedor el derecho a raw sockets; así ya no se pueden generar paquetes IP falsificados |
| `--user 1000:1000` | Ejecuta el proceso con el ID de usuario y grupo 1000 en lugar de como root |
| `example/app:1.4` | Imagen y etiqueta del contenedor (ejemplo) |

</details>

`ping` requiere `NET_RAW` en algunas imágenes; en kernels actuales suele funcionar también sin ella mediante sockets ICMP sin privilegios. En Kubernetes, esto se gestiona mediante el `securityContext` del contenedor:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - NET_RAW
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `runAsNonRoot: true` | Kubernetes no inicia el contenedor si se ejecutaría como root |
| `allowPrivilegeEscalation: false` | Impide que el proceso obtenga derechos adicionales, por ejemplo mediante programas setuid |
| `capabilities.drop: [NET_RAW]` | Retira la capability de raw socket, igual que `--cap-drop=NET_RAW` en Docker |

</details>

El Pod Security Standard `restricted` exige que los contenedores entreguen todas las capabilities (`drop: [ALL]`), por lo que cubre `NET_RAW`.

### Refuerzo de NAT

Instale las actualizaciones disponibles: kernel o firmware con la corrección de Netfilter, las actualizaciones de Windows de julio y agosto de 2026 en hosts Hyper-V con red NAT, y FreeBSD a partir de 15.0.0. Puede consultar si el kernel de su distribución contiene la corrección para CVE-2026-63913 en el changelog del paquete del kernel.

En routers Linux y hosts de contenedores, también se puede desactivar el modo laxo. Después, Netfilter solo incorporará a la tabla conexiones TCP con handshake completo.

```bash
sysctl net.netfilter.nf_conntrack_tcp_loose
sudo sysctl -w net.netfilter.nf_conntrack_tcp_loose=0
echo "net.netfilter.nf_conntrack_tcp_loose = 0" | sudo tee /etc/sysctl.d/90-natjack.conf
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `sysctl <schlüssel>` | Muestra el valor actual; `1` significa que el modo laxo está activo (predeterminado) |
| `-w` | Escribe el valor inmediatamente en el kernel en ejecución, sin reiniciar |
| `nf_conntrack_tcp_loose=0` | Desactiva la aceptación de conexiones cuyo handshake no se ha visto |
| `tee /etc/sysctl.d/90-natjack.conf` | Guarda la configuración de forma permanente para que sobreviva a un reinicio |

</details>

El modo laxo existe por un motivo: tras reiniciar el router o un failover en un clúster con `conntrackd`, las conexiones existentes se interrumpen sin él. Compruébelo antes de realizar el cambio, sobre todo en firewalls de alta disponibilidad. El white paper señala expresamente que varias medidas equilibran seguridad, disponibilidad y compatibilidad de protocolos.

Otras configuraciones, siempre que su producto las ofrezca:

- **Límite de conexiones por cliente:** El white paper recomienda un límite de aproximadamente 10 000 conexiones simultáneas por cliente. En firewalls basados en pf como OPNsense y pfSense, la configuración se encuentra en las opciones avanzadas de la regla de firewall (pf: `max-src-states`).
- **Desactivar la preservación de puertos y el mapeo independiente del extremo:** Dificulta la determinación de puertos. En pf, una regla NAT sin `static-port` sustituye el puerto de origen de forma aleatoria. Tenga en cuenta que algunas aplicaciones VoIP y peer-to-peer dependen de asignaciones fijas.
- **Aleatorización de ISN:** En pf, mediante `modulate state` en lugar de `keep state`. Debe evitar las reglas con `flags any`, porque permiten conexiones sin un handshake válido.

### Supervisión

El white paper menciona varios indicios de intentos de ataque: una tabla NAT llena o casi llena, ráfagas de paquetes de un cliente sobre un amplio rango de puertos de origen, la misma dirección IP en dos direcciones MAC o una dirección MAC con varias direcciones IP, ráfagas de paquetes RST con números de secuencia cambiantes, números de secuencia muy fuera de la ventana TCP y paquetes con TTL sospechosamente bajo. En Linux, el nivel de ocupación de la tabla se muestra comparando dos valores:

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count /proc/sys/net/netfilter/nf_conntrack_max
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `nf_conntrack_count` | Número de entradas actualmente ocupadas en la tabla |
| `nf_conntrack_max` | Tamaño máximo de la tabla; depende de la memoria RAM (con 4 GB o más, 262 144 entradas) |

</details>

Si el primer valor se mantiene permanentemente cerca del segundo, es motivo para investigarlo, independientemente de NatJack. Muchos firewalls muestran el mismo indicador como "State Table" o "Sessions" en el panel y pueden alertar al superar un umbral.

## Contexto

NatJack explota supuestos presentes desde hace décadas en las implementaciones NAT, y no un único error de programación. Por eso tampoco existe un parche que elimine por completo la clase de ataque: Linux y Windows han aumentado la dificultad, AWS ha mejorado el servicio en el servidor y muchos dispositivos permanecen sin actualización. Las medidas sostenibles se encuentran en la propia arquitectura: protocolos cifrados también en la red interna y ningún dispositivo NAT compartido para sistemas en los que confía en distinto grado. Quien haya aplicado ambas medidas puede afrontar NatJack con el ciclo habitual de parches.

## Fuentes

1.  [NatJack (natjack.io)](https://natjack.io/): página del proyecto del investigador con matriz de parches por producto, lista de CVE y recomendaciones; estado de la matriz al 19 de agosto de 2026.

2.  [NatJack White Paper 1.0 (PDF)](https://natjack.io/natjack_white_paper_1.0.pdf): Malcolm Stagg, SODIUM-24, LLC, agosto de 2026; descripción técnica de las cinco vulnerabilidades, valoraciones CVSS, mitigaciones e indicadores.

3.  [Black Hat USA 2026: Breaking Trust Boundaries](https://blackhat.com/us-26/briefings/schedule/#breaking-trust-boundaries-exploiting-design-assumptions-in-network-infrastructure-53311): ponencia del 6 de agosto de 2026 con la que la clase de ataque se hizo pública.

4.  [Microsoft: CVE-2026-56181](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56181): Windows NAT, variante downstream en Hyper-V; actualización de julio de 2026.

5.  [Microsoft: CVE-2026-56179](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-56179): Windows NAT, variante upstream en Hyper-V; actualización de agosto de 2026, mitigación que se activa mediante el registro.

6.  [Kernel Linux: commit de Netfilter para CVE-2026-63913](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/netfilter/nf_conntrack_proto_tcp.c?id=2bb6d82b586ea5a4cb73bbdd6b7432e96096bc77): mitigación parcial en el seguimiento TCP de Netfilter.

7.  [FreeBSD: commit de pf](https://github.com/freebsd/freebsd-src/commit/9485d22286603ef2ac2e2009dd7dfcc6a59e2283): cambio adoptado desde OpenBSD, que en FreeBSD 15.0.0 dificulta considerablemente la mayoría de las variantes.

8.  [eero: Protecting eero customers from NatJack](https://eero.com/blog/articles/protecting-eero-customers-from-natjack): comunicado del fabricante sobre sus propios dispositivos.

9.  [AWS: Troubleshoot NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html): indicaciones sobre la utilización de puertos tras el refuerzo en el servidor, al que AWS remite en su declaración.

10.  [Documentación del kernel Linux: Netfilter Conntrack Sysctl](https://docs.kernel.org/networking/nf_conntrack-sysctl.html): descripción de `nf_conntrack_tcp_loose`, `nf_conntrack_max` y los demás parámetros de Conntrack.
