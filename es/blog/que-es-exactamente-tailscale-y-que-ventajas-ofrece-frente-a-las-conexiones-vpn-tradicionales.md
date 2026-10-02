---
title: "¿Qué es exactamente Tailscale y qué ventajas ofrece frente a las conexiones VPN tradicionales?"
navTitle: "Tailscale frente a VPN"
description: "Tailscale crea una VPN en malla basada en WireGuard, en la que los dispositivos se conectan directamente en lugar de hacerlo a través de un concentrador VPN central. Cómo interactúan el servidor de coordinación, el recorrido de NAT y los relés DERP, cuáles son las ventajas frente a las VPN IPsec y SSL, y qué dependencias y límites debe conocer antes de utilizarlo."
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
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/es/blog/que-es-exactamente-tailscale-y-que-ventajas-ofrece-frente-a-las-conexiones-vpn-tradicionales
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:05:58.009Z
translationReview: automatic
---

# ¿Qué es exactamente Tailscale y qué ventajas ofrece frente a las conexiones VPN tradicionales?

Tailscale es un servicio VPN que conecta dispositivos a una red privada, la denominada Tailnet. Técnicamente se basa en WireGuard. La diferencia con una VPN empresarial clásica reside en la topología: los dispositivos establecen sus túneles cifrados directamente entre sí (malla). No se necesita un concentrador VPN central por el que circule todo el tráfico. Solo la administración permanece centralizada: un servidor de coordinación distribuye claves públicas, direcciones y reglas de acceso, pero no ve los datos de uso.

Para empezar, basta con una cuenta en un proveedor de identidad (Microsoft, Google, GitHub, Apple o un proveedor OIDC) y el cliente en cada dispositivo. Por regla general, no son necesarias aperturas de puertos en el firewall. Por ello, Tailscale está muy extendido tanto para redes domésticas como para el acceso remoto a servidores y sistemas de clientes.

## Cómo funciona una VPN tradicional

Una VPN clásica de acceso remoto funciona según el principio hub-and-spoke. En el perímetro de la red corporativa hay una puerta de enlace VPN (firewall o appliance) accesible desde Internet. El cliente del portátil establece un túnel hacia esta puerta de enlace; son habituales IPsec/IKEv2 (UDP 500 y 4500), variantes de VPN SSL mediante TCP 443 u OpenVPN. Tras iniciar sesión, el dispositivo recibe una dirección de un pool y rutas hacia las redes internas.

Este modelo ha demostrado su eficacia durante décadas, pero presenta características estructurales que generan esfuerzo en las operaciones actuales:

- **Puerta de enlace accesible públicamente:** El concentrador VPN debe ser accesible desde Internet y, por tanto, es un objetivo de ataque preferente. Las vulnerabilidades en appliances VPN se han explotado activamente de forma repetida en los últimos años; en enero de 2024, la agencia estadounidense CISA incluso ordenó, mediante la Directiva de Emergencia 24-01, desconectar de la red las puertas de enlace Ivanti afectadas.
- **Punto único de fallo y cuello de botella:** Todo el tráfico pasa por la puerta de enlace. Si falla o se agota el ancho de banda, afecta a todos los usuarios al mismo tiempo.
- **Desvíos (hairpinning):** Si dos empleados en teletrabajo acceden al mismo servidor en la nube, el tráfico circula primero al centro de datos y desde allí vuelve a salir.
- **Permisos de acceso amplios:** Tras establecer la conexión, el dispositivo suele tener acceso a todo un segmento de red. Es posible establecer reglas detalladas por usuario y servicio, pero rara vez se mantienen de forma coherente.
- **Esfuerzo de sitio a sitio:** Cada ubicación adicional necesita su propio túnel con parámetros coordinados (propuestas de fase 1/fase 2, claves precompartidas o certificados, direcciones IP públicas fijas).

## Cómo está estructurado Tailscale

Tailscale separa el plano de control del plano de datos. El plano de control es el servidor de coordinación, que Tailscale opera como servicio en la nube. El plano de datos son los túneles WireGuard entre los dispositivos.

| Componente | Función |
|---|---|
| Cliente (`tailscaled`) | Genera el par de claves localmente, establece túneles WireGuard con los demás nodos y aplica localmente las reglas de acceso |
| Servidor de coordinación | Autentica dispositivos mediante el proveedor de identidad, distribuye claves públicas, direcciones, configuración DNS y la política a todos los nodos |
| Relés DERP | Reenvían paquetes cifrados cuando no puede establecerse una conexión directa |
| Relés de pares | Dispositivos propios en la Tailnet que actúan como relé con mayor rendimiento; se priorizan frente a DERP |

La clave privada de un dispositivo nunca sale de él. El servidor de coordinación solo conoce las claves públicas y, por ello, no puede descifrar el tráfico. Cada dispositivo recibe una dirección fija del rango `100.64.0.0/10` (el espacio de direcciones para NAT de grado de operador), así como una dirección IPv6 de `fd7a:115c:a1e0::/48`. Mediante MagicDNS, los dispositivos también son accesibles por su nombre, por ejemplo `nas` o `nas.tailnet-name.ts.net`.

### Recorrido de NAT: por qué no se necesitan aperturas de puertos

La mayoría de los dispositivos se encuentran detrás de un router NAT o un firewall y no son accesibles directamente desde el exterior. Tailscale lo resuelve con recorrido de NAT: ambos extremos determinan mediante STUN su dirección pública y el puerto asignado, intercambian esta información a través del servidor de coordinación y envían simultáneamente paquetes UDP entre sí. Los paquetes salientes abren una entrada de estado en ambos firewalls, a través de la cual posteriormente entran los paquetes del otro extremo (UDP Hole Punching).

Si esto no funciona, por ejemplo, con firewalls restrictivos que bloquean UDP saliente o con determinadas formas de NAT de grado de operador, el tráfico pasa por un relé DERP mediante HTTPS. También en este caso permanece cifrado de extremo a extremo con WireGuard; el relé solo ve paquetes cifrados. El precio es una latencia mayor y un rendimiento menor. Los relés de pares reducen este inconveniente al hacer que un dispositivo propio con buena conectividad asuma el reenvío.

## Las ventajas frente a una VPN tradicional

| Criterio | VPN tradicional | Tailscale |
|---|---|---|
| Topología | Hub-and-spoke mediante una puerta de enlace central | Malla, conexiones directas entre dispositivos |
| Puertos entrantes | La puerta de enlace debe ser accesible desde Internet | No se necesitan aperturas de puertos entrantes |
| Autenticación | Cuentas locales, RADIUS, certificados, a menudo MFA independiente | Inicio de sesión mediante el proveedor de identidad existente, incluido su MFA |
| Permisos de acceso | A menudo por segmento de red | Por usuario, grupo, dispositivo y puerto en una política central |
| Nueva ubicación | Túnel de sitio a sitio con parámetros coordinados | Instalar cliente o router de subred |
| Fallo del centro | Ya no hay acceso para nadie | Las conexiones existentes continúan; los dispositivos nuevos y los cambios de política esperan |
| Protocolo | IPsec, VPN SSL, OpenVPN | WireGuard |

### Menor superficie de ataque

Dado que los clientes establecen sus conexiones de salida, ningún dispositivo necesita un puerto abierto en Internet. Un servidor que solo deba ser accesible mediante la Tailnet puede vincular sus servicios exclusivamente a la interfaz de Tailscale. Así no será visible para los escáneres de puertos de Internet. WireGuard, con unas 4000 líneas de código de kernel, es considerablemente más pequeño que las implementaciones típicas de VPN IPsec o SSL y utiliza un conjunto fijo de procedimientos modernos (Curve25519, ChaCha20-Poly1305, BLAKE2s). No existe negociación de conjuntos de cifrado, como la que provoca regularmente configuraciones erróneas en IPsec.

### Identidad en lugar de dirección de red

Cada dispositivo de la Tailnet está vinculado a un usuario o a una etiqueta. El inicio de sesión se realiza mediante el proveedor de identidad que ya utiliza; por tanto, MFA y el acceso condicional de Microsoft Entra ID también se aplican al acceso de red. Los dispositivos nuevos solo pueden incorporarse a la Tailnet después de iniciar sesión correctamente en el proveedor de identidad, y cada clave de dispositivo caduca de forma predeterminada después de 180 días. Si un empleado deja la empresa, bloquee su cuenta en el proveedor de identidad; con el aprovisionamiento SCIM (a partir del plan Standard), el usuario se desactiva automáticamente en la Tailnet y sus dispositivos pierden el acceso.

### Reglas de acceso según el principio de privilegio mínimo

De forma predeterminada, en una Tailnet nueva cada dispositivo puede alcanzar a cualquier otro. Para uso en producción, defina los permisos de acceso en un archivo de política central (HuJSON). El siguiente ejemplo permite al grupo de administradores usar SSH y HTTPS en todos los servidores con la etiqueta `tag:server`, y al resto de usuarios solo HTTPS en el servidor de intranet:

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
| `groups` | Define grupos de usuarios; los miembros se indican mediante su dirección de inicio de sesión en el proveedor de identidad |
| `tagOwners` | Determina quién puede asignar una etiqueta a los dispositivos; los dispositivos etiquetados no pertenecen a ningún usuario, sino a la etiqueta |
| `grants` | Lista de conexiones permitidas; todo lo que no esté expresamente permitido se bloquea |
| `src` | Origen de la conexión: usuario, grupo, etiqueta o `autogroup:member` (todos los usuarios de la Tailnet) |
| `dst` | Destino de la conexión: etiqueta, alias de host, dispositivo o subred |
| `ip` | Protocolos y puertos permitidos, por ejemplo `tcp:22` o `*` para todo |
| `hosts` | Nombres de alias para direcciones o subredes de la Tailnet utilizados en las reglas |

</details>

Cada cliente aplica las reglas localmente. Un paquete que la política no permite se descarta ya en el dispositivo de destino. El servidor de coordinación solo distribuye la política.

### Conexiones directas en lugar de desvíos

Como los túneles existen directamente entre los dispositivos, el tráfico sigue la ruta más corta. Dos dispositivos en la misma oficina se comunican localmente, y un portátil en teletrabajo accede directamente a un servidor en la nube. Esto reduce la latencia y descarga la conexión a Internet de la ubicación principal.

### Conectar redes existentes

No todos los dispositivos pueden ejecutar un cliente de Tailscale, como impresoras, sistemas NAS antiguos o controladores industriales. Para estos casos, un router de subred asume la función de puerta de enlace: un servidor Linux en la red de destino anuncia (advertise) la subred local en la Tailnet, y los dispositivos autorizados acceden a ella a través de él.

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
| `curl -fsSL …/install.sh \| sh` | Descarga el script oficial de instalación y configura el repositorio de paquetes de la distribución |
| `net.ipv4.ip_forward = 1` | Permite al kernel de Linux reenviar paquetes entre interfaces; sin esta configuración no funciona ningún router de subred |
| `sysctl -p <datei>` | Carga la configuración inmediatamente, sin reiniciar |
| `tailscale up` | Registra el dispositivo en la Tailnet; en la primera ejecución aparece un enlace de inicio de sesión |
| `--advertise-routes=<subnetz>` | Anuncia la subred indicada en la Tailnet; varias subredes se separan con comas |
| `--advertise-tags=<tag>` | Asigna al dispositivo una etiqueta a la que se refieren las reglas de acceso |

</details>

Posteriormente, la ruta anunciada debe aprobarse en la consola de administración, salvo que se haya configurado una aprobación automática (`autoApprovers`) en la política. De forma análoga, un dispositivo puede operar como nodo de salida con `--advertise-exit-node`. Los clientes que seleccionen este nodo de salida enrutarán todo su tráfico de Internet a través de él, lo que equivale al modo de túnel completo de una VPN clásica.

### Menos esfuerzo operativo

El servidor de coordinación gestiona la rotación de claves, la asignación de direcciones, DNS y el enrutamiento. Una nueva ubicación necesita un router de subred con acceso a Internet, pero ninguna dirección IP pública fija ni coordinación de parámetros IPsec con el otro extremo. Para el acceso remoto a servidores, también están disponibles Tailscale SSH (inicio de sesión mediante identidad de Tailnet sin claves SSH distribuidas) y `tailscale serve` (publicación de un servicio web local en la Tailnet).

## Límites y dependencias

Tailscale sustituye la puerta de enlace VPN clásica, pero desplaza parte de la responsabilidad a un servicio externo. Debe revisar estos puntos antes de implantarlo.

| Tema | Aspectos a tener en cuenta |
|---|---|
| Dependencia del proveedor | El plano de control es un servicio en la nube de Tailscale Inc. En caso de interrupción, las conexiones existentes continúan, pero los nuevos dispositivos, inicios de sesión y cambios de política no son posibles hasta que se restablezca el servicio |
| Metadatos | Los nombres de los dispositivos, direcciones de Tailnet, direcciones IP públicas, cuentas de usuario y momentos de conexión se procesan por el proveedor; los datos de uso no |
| Confianza en la distribución de claves | El servidor de coordinación determina qué claves públicas acepta un dispositivo. Tailnet Lock exige además una firma de dispositivos propios de confianza |
| Código fuente | El núcleo del cliente es de código abierto (BSD-3-Clause), el servidor de coordinación no. Headscale es una alternativa de código abierto autoalojada con funcionalidad reducida |
| Conflictos de direcciones | `100.64.0.0/10` también lo utilizan algunos proveedores para NAT de grado de operador y otros productos VPN; los solapamientos provocan problemas de enrutamiento |
| Coexistencia con otros clientes VPN | Un segundo cliente VPN con túnel completo puede redirigir el tráfico de Tailscale. El rango `100.64.0.0/10` y el servicio de Tailscale deben excluirse allí del túnel |
| Rendimiento mediante relés | Si no se establece una conexión directa, el rendimiento a través de DERP disminuye notablemente; `tailscale netcheck` muestra si UDP saliente funciona |
| No sustituye el filtrado web | Tailscale regula el acceso a recursos internos. No se encarga del filtrado de contenido ni de la inspección del tráfico de Internet |

Para las empresas en Suiza se aplica la Ley Federal de Protección de Datos revisada (revDSG). Dado que el proveedor trata datos personales de los usuarios (cuentas, dispositivos, metadatos de conexión), incluya Tailscale en el registro de actividades de tratamiento si su empresa debe llevar uno (a partir de 250 empleados o en tratamientos de alto riesgo). Revise el contrato de encargado del tratamiento (Data Processing Addendum) del proveedor y la base jurídica de la comunicación al extranjero conforme al art. 16 de la revDSG, por ejemplo, una certificación bajo el Swiss-U.S. Data Privacy Framework o cláusulas contractuales tipo. Si se excluye el tratamiento de datos por parte del proveedor, Headscale sigue siendo una alternativa como plano de control autogestionado.

> **Nota sobre la UE:** Para las sucursales en la UE se aplican el RGPD y sus normas sobre transferencias a terceros países (art. 44 y siguientes del RGPD). La evaluación es similar en cuanto al contenido; allí es determinante el EU-U.S. Data Privacy Framework.

## Costes

El plan Personal es gratuito e incluye hasta seis usuarios, cualquier cantidad de dispositivos de usuario y 50 dispositivos etiquetados (situación a octubre de 2026). Para uso empresarial, el plan Standard cuesta 8 USD y Premium 18 USD por usuario y mes. Premium añade, entre otras cosas, Network Flow Logs, streaming de registros y opciones avanzadas para Tailscale SSH. Enterprise se ofrece de forma individualizada.

## Para quién es adecuado Tailscale

Tailscale es especialmente adecuado cuando los usuarios y los recursos están distribuidos: teletrabajo, servidores en la nube de varios proveedores, pequeñas ubicaciones externas sin dirección IP fija o mantenimiento remoto en clientes. Para pequeñas y medianas empresas, puede sustituir por completo una puerta de enlace VPN. En entornos más grandes, suele ejecutarse en paralelo con la VPN existente, por ejemplo, para el acceso administrativo a servidores, donde las reglas de acceso detalladas aportan el mayor beneficio.

Es menos adecuado cuando los requisitos exigen una infraestructura totalmente autogestionada y Headscale no cubre la funcionalidad necesaria, o cuando todo el tráfico de Internet debe filtrarse centralmente. En este caso, sigue siendo necesaria una Secure Web Gateway que complemente Tailscale.

Para probarlo, basta con instalar el cliente en dos dispositivos e iniciar sesión con la misma cuenta. Con `tailscale status` verá después todos los dispositivos de la Tailnet; con `tailscale ping <gerät>` comprobará si se utiliza una conexión directa o un relé.

## Fuentes

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): arquitectura con servidor de coordinación, malla WireGuard y aplicación local de las reglas.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): explicación detallada de STUN, UDP Hole Punching y los casos en que se necesita un relé.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): función y cifrado de los servidores de relé.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): dispositivos propios como relé con prioridad sobre DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): sintaxis de las reglas de acceso en el archivo de política.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): configuración de routers de subred, incluido el reenvío IP y la aprobación de rutas.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): enrutar todo el tráfico de Internet a través de un dispositivo de la Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): firma de dispositivos nuevos mediante nodos propios de confianza.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): planes y límites, consultado el 1 de octubre de 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): diseño del protocolo y procedimientos criptográficos utilizados.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): código fuente del cliente bajo BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementación de código abierto del servidor de coordinación para operación propia.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): orden de desconectar las puertas de enlace VPN Ivanti vulnerables en enero de 2024.

14.  [Fedlex: Ley Federal de Protección de Datos (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 sobre la comunicación de datos personales al extranjero.
