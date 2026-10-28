---
title: "¿Qué es exactamente Tailscale y qué ventajas ofrece frente a las conexiones VPN tradicionales?"
navTitle: "Tailscale frente a VPN"
description: "Tailscale crea una VPN de malla basada en WireGuard, en la que los dispositivos se conectan directamente en lugar de hacerlo a través de un concentrador VPN central. Cómo interactúan el Coordination Server, el NAT traversal y los relés DERP, cuáles son las ventajas frente a las VPN IPsec y SSL, y qué dependencias y limitaciones debe conocer antes de utilizarlo."
date: "2026-10-01"
kategorie: "VPN y acceso remoto"
timeToRead: "11 min de lectura"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "que-es-exactamente-tailscale-y-que-ventajas-ofrece-frente-a-las-conexiones-vpn-tradicionales"
translationId: "article-91fddf1e0239f5c4"
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:22:23.649Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/que-es-exactamente-tailscale-y-que-ventajas-ofrece-frente-a-las-conexiones-vpn-tradicionales
---

# ¿Qué es exactamente Tailscale y qué ventajas ofrece frente a las conexiones VPN tradicionales?

Tailscale es un servicio VPN que conecta dispositivos a una red privada, la denominada Tailnet. Técnicamente se basa en WireGuard. La diferencia con una VPN empresarial clásica está en la topología: los dispositivos establecen sus túneles cifrados directamente entre sí (malla). No se necesita un concentrador VPN central por el que pase todo el tráfico. Solo la administración permanece centralizada: un Coordination Server distribuye claves públicas, direcciones y reglas de acceso, pero no ve los datos de usuario.

Para empezar, basta con una cuenta en un Identity Provider (Microsoft, Google, GitHub, Apple o un proveedor OIDC) y el cliente en cada dispositivo. Por regla general, no se necesitan redirecciones de puertos en el firewall. Por ello, Tailscale se utiliza tanto para redes domésticas como para el acceso remoto a servidores y sistemas de clientes.

## Cómo funciona una VPN tradicional

Una VPN clásica de acceso remoto funciona según el principio hub-and-spoke. En el perímetro de la red corporativa hay una puerta de enlace VPN (firewall o appliance) accesible desde Internet. El cliente del portátil establece un túnel con esta puerta de enlace; son habituales IPsec/IKEv2 (UDP 500 y 4500), variantes de VPN SSL mediante TCP 443 u OpenVPN. Tras iniciar sesión, el dispositivo recibe una dirección de un pool y rutas hacia las redes internas.

Este modelo ha demostrado su eficacia durante décadas, pero presenta características estructurales que hoy generan trabajo operativo:

- **Puerta de enlace accesible públicamente:** El concentrador VPN debe ser accesible desde Internet y, por tanto, es un objetivo de ataque prioritario. En los últimos años se han explotado activamente vulnerabilidades en appliances VPN de forma reiterada; en enero de 2024, la agencia estadounidense CISA incluso ordenó, mediante la Emergency Directive 24-01, desconectar de la red las puertas de enlace Ivanti afectadas.
- **Punto único de fallo y cuello de botella:** Todo el tráfico pasa por la puerta de enlace. Si falla o se agota el ancho de banda, afecta a todos los usuarios a la vez.
- **Desvíos (hairpinning):** Si dos empleados que trabajan desde casa acceden al mismo servidor en la nube, el tráfico pasa primero por el centro de datos y vuelve a salir desde allí.
- **Derechos de acceso amplios:** Tras establecer la conexión, el dispositivo suele tener acceso a todo un segmento de red. Las reglas granulares por usuario y servicio son posibles, pero rara vez se mantienen de forma sistemática.
- **Esfuerzo para site-to-site:** Cada ubicación adicional necesita su propio túnel con parámetros coordinados (proposals de fase 1/fase 2, claves precompartidas o certificados, direcciones IP públicas fijas).

## Estructura de Tailscale

Tailscale separa el plano de control del plano de datos. El plano de control es el Coordination Server, que Tailscale opera como servicio en la nube. El plano de datos son los túneles WireGuard entre los dispositivos.

| Componente | Función |
|---|---|
| Cliente (`tailscaled`) | Genera el par de claves localmente, establece túneles WireGuard con los demás nodos y aplica localmente las reglas de acceso |
| Coordination Server | Autentica dispositivos mediante el Identity Provider, distribuye claves públicas, direcciones, configuraciones DNS y la política a todos los nodos |
| Relés DERP | Reenvían paquetes cifrados cuando no se puede establecer una conexión directa |
| Peer Relays | Dispositivos propios en la Tailnet que sirven como relé con mayor rendimiento; tienen prioridad sobre DERP |

La clave privada de un dispositivo nunca sale de este. El Coordination Server solo conoce las claves públicas y, por ello, no puede descifrar el tráfico. Cada dispositivo recibe una dirección fija del rango `100.64.0.0/10` (el espacio de direcciones para Carrier-Grade NAT), así como una dirección IPv6 de `fd7a:115c:a1e0::/48`. Mediante MagicDNS, los dispositivos también son accesibles por su nombre, por ejemplo `nas` o `nas.tailnet-name.ts.net`.

### NAT traversal: por qué no se necesitan redirecciones de puertos

La mayoría de los dispositivos se encuentran detrás de un router NAT o un firewall y no son accesibles directamente desde el exterior. Tailscale resuelve esto con NAT traversal: ambos extremos determinan mediante STUN su dirección pública y el puerto asignado, intercambian esta información a través del Coordination Server y envían simultáneamente paquetes UDP entre sí. Los paquetes salientes abren una entrada de estado en ambos firewalls, a través de la cual entran posteriormente los paquetes del otro extremo (UDP hole punching).

Si esto no funciona, por ejemplo, con firewalls restrictivos que bloquean UDP saliente o con determinadas formas de Carrier-Grade NAT, el tráfico pasa por un relé DERP mediante HTTPS. También en este caso permanece cifrado de extremo a extremo con WireGuard; el relé solo ve paquetes cifrados. El coste es una latencia mayor y un menor rendimiento. Los Peer Relays reducen esta desventaja al hacer que un dispositivo propio con buena conectividad se encargue del reenvío.

## Las ventajas frente a una VPN tradicional

| Criterio | VPN tradicional | Tailscale |
|---|---|---|
| Topología | Hub-and-spoke mediante una puerta de enlace central | Malla, conexiones directas entre dispositivos |
| Puertos entrantes | La puerta de enlace debe ser accesible desde Internet | No se necesitan redirecciones de puertos entrantes |
| Autenticación | Cuentas locales, RADIUS, certificados, a menudo MFA independiente | Inicio de sesión mediante el Identity Provider existente, incluido su MFA |
| Derechos de acceso | A menudo por segmento de red | Por usuario, grupo, dispositivo y puerto en una política central |
| Nueva ubicación | Túnel site-to-site con parámetros coordinados | Instalar cliente o Subnet Router |
| Fallo del centro | Se pierde el acceso para todos | Las conexiones existentes continúan; los dispositivos nuevos y los cambios de política esperan |
| Protocolo | IPsec, VPN SSL, OpenVPN | WireGuard |

### Menor superficie de ataque

Como los clientes establecen sus conexiones de salida, ningún dispositivo necesita un puerto abierto en Internet. Un servidor que solo deba ser accesible a través de la Tailnet puede vincular sus servicios exclusivamente a la interfaz de Tailscale. De este modo, no es visible para los escáneres de puertos de Internet. WireGuard, con unas 4000 líneas de código de kernel, es considerablemente más pequeño que las implementaciones típicas de IPsec o VPN SSL y utiliza un conjunto fijo de métodos modernos (Curve25519, ChaCha20-Poly1305, BLAKE2s). No existe la negociación de suites de cifrado que en IPsec provoca regularmente configuraciones erróneas.

### Identidad en lugar de dirección de red

Cada dispositivo de la Tailnet está vinculado a un usuario o a una etiqueta. El inicio de sesión se realiza mediante el Identity Provider que ya utiliza; por tanto, MFA y Conditional Access de Microsoft Entra ID también se aplican al acceso a la red. Los nuevos dispositivos solo pueden incorporarse a la Tailnet tras iniciar sesión correctamente en el Identity Provider, y cada clave de dispositivo caduca por defecto después de 180 días. Si un empleado abandona la empresa, bloquee su cuenta en el Identity Provider; con aprovisionamiento SCIM (a partir del plan Standard), el usuario se desactiva automáticamente en la Tailnet y sus dispositivos pierden el acceso.

### Reglas de acceso según el principio de mínimo privilegio

Por defecto, en una Tailnet nueva cada dispositivo puede acceder a cualquier otro. Para uso productivo, defina los derechos de acceso en un archivo de política central (HuJSON). El siguiente ejemplo permite al grupo de administradores SSH y HTTPS en todos los servidores con la etiqueta `tag:server`, y a todos los demás usuarios solo HTTPS en el servidor de intranet:

```json
{
  "groups": {
    "group:admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:server": ["group:admins"]
  },
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "ip":  ["tcp:22", "tcp:443"]
    },
    {
      "src": ["autogroup:member"],
      "dst": ["intranet"],
      "ip":  ["tcp:443"]
    }
  ],
  "hosts": {
    "intranet": "100.101.102.103"
  }
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `groups` | Define grupos de usuarios; los miembros se indican mediante su dirección de inicio de sesión en el Identity Provider |
| `tagOwners` | Establece quién puede asignar una etiqueta a dispositivos; los dispositivos etiquetados no pertenecen a ningún usuario, sino a la etiqueta |
| `grants` | Lista de conexiones permitidas; todo lo que no esté expresamente permitido se bloquea |
| `src` | Origen de la conexión: usuario, grupo, etiqueta o `autogroup:member` (todos los usuarios de la Tailnet) |
| `dst` | Destino de la conexión: etiqueta, alias de host, dispositivo o subred |
| `ip` | Protocolos y puertos permitidos, por ejemplo `tcp:22` o `*` para todo |
| `hosts` | Nombres de alias para direcciones o subredes de la Tailnet utilizados en las reglas |

</details>

Las reglas se aplican localmente en cada cliente. Un paquete no permitido por la política ya es descartado por el dispositivo de destino. El Coordination Server solo distribuye la política.

### Conexiones directas en lugar de desvíos

Dado que los túneles existen directamente entre los dispositivos, el tráfico toma el camino más corto. Dos dispositivos en la misma oficina se comunican localmente, y un portátil en la oficina doméstica accede directamente a un servidor en la nube. Esto reduce la latencia y descarga la conexión a Internet de la ubicación principal.

### Conectar redes existentes

No todos los dispositivos pueden ejecutar un cliente Tailscale, como impresoras, sistemas NAS antiguos o controles industriales. En estos casos, un Subnet Router asume el papel de puerta de enlace: un servidor Linux en la red de destino anuncia la subred local en la Tailnet (advertise), y los dispositivos autorizados acceden a ella a través de este.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
echo 'net.ipv4.ip_forward = 1' | \
  sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up \
  --advertise-routes=192.168.10.0/24 \
  --advertise-tags=tag:server
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Descarga el script de instalación oficial y configura el repositorio de paquetes de la distribución |
| `net.ipv4.ip_forward = 1` | Permite al kernel Linux reenviar paquetes entre interfaces; sin este ajuste no funciona ningún Subnet Router |
| `sysctl -p <datei>` | Carga el ajuste inmediatamente, sin reiniciar |
| `tailscale up` | Registra el dispositivo en la Tailnet; en la primera ejecución aparece un enlace de inicio de sesión |
| `--advertise-routes=<subnetz>` | Anuncia la subred indicada en la Tailnet; varias subredes se separan con comas |
| `--advertise-tags=<tag>` | Asigna al dispositivo una etiqueta a la que se refieren las reglas de acceso |

</details>

La ruta anunciada debe aprobarse posteriormente en la consola de administración, salvo que se haya configurado una aprobación automática (`autoApprovers`) en la política. De forma análoga, un dispositivo puede utilizarse como Exit Node con `--advertise-exit-node`. Los clientes que seleccionen este Exit Node reenviarán entonces todo su tráfico de Internet a través de él, lo que corresponde al modo full tunnel de una VPN clásica.

### Menor esfuerzo operativo

El Coordination Server gestiona la rotación de claves, la asignación de direcciones, DNS y el enrutamiento. Una nueva ubicación necesita un Subnet Router con acceso a Internet, pero no una dirección IP pública fija ni coordinación de parámetros IPsec con el otro extremo. Para el acceso remoto a servidores, también están disponibles Tailscale SSH (inicio de sesión mediante la identidad de la Tailnet sin claves SSH distribuidas) y `tailscale serve` (publicación de un servicio web local en la Tailnet).

## Límites y dependencias

Tailscale sustituye la puerta de enlace VPN clásica, pero traslada parte de la responsabilidad a un servicio externo. Debe revisar estos puntos antes de implantarlo.

| Tema | Aspectos a tener en cuenta |
|---|---|
| Dependencia del proveedor | El plano de control es un servicio en la nube de Tailscale Inc. En caso de fallo, las conexiones existentes continúan, pero los dispositivos nuevos, los inicios de sesión y los cambios de política no son posibles hasta su restablecimiento |
| Metadatos | El proveedor procesa nombres de dispositivos, direcciones de Tailnet, direcciones IP públicas, cuentas de usuario y momentos de conexión; no procesa los datos de usuario |
| Confianza en la distribución de claves | El Coordination Server determina qué claves públicas acepta un dispositivo. Tailnet Lock exige además una firma de dispositivos propios de confianza |
| Código fuente | El núcleo del cliente es de código abierto (BSD-3-Clause), pero el Coordination Server no lo es. Headscale es una alternativa de código abierto autoalojada con funcionalidades reducidas |
| Conflictos de direcciones | `100.64.0.0/10` también lo utilizan algunos proveedores para Carrier-Grade NAT y otros productos VPN; los solapamientos provocan problemas de enrutamiento |
| Coexistencia con otros clientes VPN | Un segundo cliente VPN con full tunnel puede redirigir el tráfico de Tailscale. El rango `100.64.0.0/10` y el servicio Tailscale deben excluirse allí del túnel |
| Rendimiento mediante relés | Si no se establece una conexión directa, el rendimiento disminuye notablemente a través de DERP; `tailscale netcheck` indica si UDP funciona hacia el exterior |
| No sustituye el filtrado web | Tailscale regula el acceso a recursos internos. No realiza filtrado de contenidos ni inspección del tráfico de Internet |

Para empresas en Suiza se aplica la Ley Federal de Protección de Datos revisada (revDSG). Dado que el proveedor procesa datos personales de los usuarios (cuentas, dispositivos, metadatos de conexión), incluya Tailscale en el registro de actividades de tratamiento si su empresa debe llevar uno (a partir de 250 empleados o en caso de tratamientos de alto riesgo). Revise el contrato de encargado del tratamiento (Data Processing Addendum) del proveedor y la base jurídica para la comunicación al extranjero conforme al art. 16 revDSG, por ejemplo una certificación bajo el Swiss-U.S. Data Privacy Framework o cláusulas contractuales tipo. Si se excluye el tratamiento de datos por parte del proveedor, Headscale sigue siendo una opción como plano de control gestionado internamente.

> **Nota para la UE:** Para las sucursales en la UE se aplican el RGPD y sus normas sobre transferencias a terceros países (art. 44 y siguientes del RGPD). La revisión es similar en cuanto al contenido; en este caso es determinante el EU-U.S. Data Privacy Framework.

## Costes

El plan Personal es gratuito e incluye hasta seis usuarios, un número ilimitado de dispositivos de usuario y 50 dispositivos etiquetados (situación en octubre de 2026). Para uso empresarial, el plan Standard cuesta 8 USD y Premium 18 USD por usuario al mes. Premium añade, entre otras cosas, Network Flow Logs, streaming de registros y opciones avanzadas para Tailscale SSH. Enterprise se ofrece de forma individualizada.

## Para quién es adecuado Tailscale

Tailscale es especialmente adecuado cuando los usuarios y los recursos están distribuidos: teletrabajo, servidores en la nube con varios proveedores, pequeñas ubicaciones externas sin dirección IP fija o mantenimiento remoto en clientes. Para pequeñas y medianas empresas, puede sustituir por completo una puerta de enlace VPN. En entornos más grandes, suele funcionar en paralelo a la VPN existente, por ejemplo para el acceso administrativo a servidores, donde las reglas de acceso granulares ofrecen el mayor beneficio.

Es menos adecuado cuando los requisitos exigen una infraestructura completamente gestionada internamente y Headscale no cubre el alcance funcional, o cuando todo el tráfico de Internet debe filtrarse de forma centralizada. En ese caso, sigue siendo necesaria una Secure Web Gateway que complemente Tailscale.

Para probarlo, basta con instalar el cliente en dos dispositivos e iniciar sesión con la misma cuenta. Con `tailscale status` verá después todos los dispositivos de la Tailnet; con `tailscale ping <gerät>` comprobará si se utiliza una conexión directa o un relé.

## Historial de cambios

- **25 de octubre de 2026:** Enlaces de fuentes actualizados a las nuevas direcciones de Tailscale.

## Fuentes

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): arquitectura con Coordination Server, malla WireGuard y aplicación local de las reglas.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): explicación detallada de STUN, UDP hole punching y los casos en los que se necesita un relé.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): función y cifrado de los servidores de relé.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): dispositivos propios como relé con prioridad sobre DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): sintaxis de las reglas de acceso en el archivo de política.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): configuración de Subnet Routers, incluido el reenvío IP y la aprobación de las rutas.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): dirigir todo el tráfico de Internet a través de un dispositivo de la Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): firma de dispositivos nuevos mediante nodos propios de confianza.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): planes y límites, consultados el 1 de octubre de 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): diseño del protocolo y métodos criptográficos utilizados.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): código fuente del cliente bajo BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementación de código abierto del Coordination Server para autoalojamiento.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): orden de desconectar las puertas de enlace VPN Ivanti vulnerables en enero de 2024.

14.  [Fedlex: Ley Federal de Protección de Datos (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 sobre la comunicación de datos personales al extranjero.
