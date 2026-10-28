---
title: "Tailscale: comparación entre Exit Node y rutas de subred, y cómo funcionan técnicamente"
navTitle: "Exit Node vs. subred"
description: "En Tailscale, un Exit Node y un router de subred son dos modos de funcionamiento relacionados, pero distintos. Un router de subred abre de forma selectiva determinados rangos de IP; un Exit Node redirige todo el tráfico de Internet a través de sí mismo. Qué significa esto en la práctica, cómo lo implementa Tailscale mediante WireGuard, la aprobación de rutas y SNAT, y cuáles son los límites de cada variante."
date: "2026-09-02"
kategorie: "Redes y VPN"
timeToRead: "11 min de lectura"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-comparacion-entre-nodos-de-salida-y-rutas-de-subred-y-como-funcionan-tecnicamente"
translationId: "article-c26cca4d635b9a04"
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:53:23.849Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/tailscale-comparacion-entre-nodos-de-salida-y-rutas-de-subred-y-como-funcionan-tecnicamente
---

# Tailscale: comparación entre Exit Node y rutas de subred, y cómo funcionan técnicamente

Un nodo de Tailscale inicialmente solo es accesible por sí mismo: mediante su dirección de Tailscale y nada más. Para que un nodo dé a otros dispositivos acceso a algo más que a sí mismo, existen dos modos de funcionamiento que a menudo se confunden: el **router de subred** y el **Exit Node**. Ambos amplían el alcance de un nodo, pero en direcciones distintas. Quien conoce la diferencia elige la variante adecuada y evita redirigir accidentalmente todo el tráfico a través de un equipo ajeno.

La versión corta: un router de subred abre **de forma selectiva determinados rangos de IP** detrás del nodo, por ejemplo la red local con un NAS y una impresora. Un Exit Node dirige **todo el tráfico de Internet** de un dispositivo a través de él, como una VPN clásica de túnel completo. Ambos se basan técnicamente en el mismo mecanismo: el anuncio de rutas. El Exit Node es, en esencia, un caso especial del router de subred en el que se anuncia la ruta predeterminada.

## Router de subred: acceso selectivo a una red

Un router de subred anuncia uno o varios rangos de IP a los que puede acceder en la red local. Los demás dispositivos de la tailnet que aceptan estas rutas pueden acceder a través de él a los dispositivos del rango anunciado, aunque no tengan Tailscale instalado. Es la forma de hacer accesible un NAS, una impresora o una interfaz de administración sin instalar un cliente VPN en cada dispositivo.

El rango se anuncia en el nodo router:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--advertise-routes=<CIDR>` | Anuncia uno o varios rangos de IP (separados por comas) que este nodo reenvía |
| `--snat-subnet-routes=false` | Reenvía sin NAT de origen, para que los dispositivos de destino vean la dirección de origen real de Tailscale; requiere una ruta de retorno en la red local |
| `--advertise-exit-node` | Forma abreviada que anuncia `0.0.0.0/0` y `::/0`, ofreciendo así el nodo como Exit Node |

</details>

El tráfico solo fluye después de que la ruta se haya **aprobado** en la administración de Tailscale. El mero anuncio no basta; este es el error más frecuente: la ruta solo aparece en la tabla de enrutamiento de los dispositivos que la aceptan tras su aprobación.

## Exit Node: todo el tráfico a través de un nodo

Un Exit Node anuncia la ruta predeterminada (`0.0.0.0/0` y `::/0`). Si un dispositivo selecciona este Exit Node, **todo** su tráfico saliente de Internet pasa por el nodo, no solo el tráfico hacia una red concreta. Esto resulta útil para salir a Internet a través de una ubicación con IP fija o para dirigir el tráfico mediante una salida de confianza en una red insegura.

La diferencia respecto a la ruta de subred es la selección en el lado del cliente: una ruta de subred se utiliza automáticamente en cuanto el dispositivo acepta la ruta y se dirige a un destino de ese rango. En cambio, un Exit Node debe seleccionarse activamente y entonces se aplica a todo el tráfico:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--exit-node=<IP oder Name>` | Selecciona un Exit Node; vacío (`--exit-node=`) lo desactiva de nuevo |
| `--exit-node-allow-lan-access` | Permite acceder a la propia red local incluso con un Exit Node activo |

</details>

Precisamente por ello, en el trabajo diario de soporte era incorrecto activar el Exit Node para acceder a un único NAS: habría redirigido todo el tráfico propio a través del equipo ajeno en lugar de abrir solo ese rango.

## Comparación

| Característica | Router de subred | Exit Node |
|---|---|---|
| Ruta anunciada | Rangos concretos, p. ej., `192.168.1.0/24` | Ruta predeterminada `0.0.0.0/0`, `::/0` |
| Uso por el cliente | Automático para destinos dentro del rango | Debe seleccionarse activamente como Exit Node |
| Alcance | Solo las redes anunciadas | Todo el tráfico de Internet |
| Aprobación en la administración | Por subred | Por separado como Exit Node |
| Finalidad habitual | Hacer accesibles servicios internos | Dirigir el tráfico saliente a través de una ubicación |

## Cómo lo implementa Tailscale técnicamente

Ambos modos de funcionamiento se basan en la misma base. Conviene separar los niveles.

**Plano de datos mediante WireGuard.** Cada nodo tiene un par de claves de WireGuard. El tráfico real entre dos nodos circula directamente como paquetes cifrados de WireGuard mediante UDP, cuando es posible entre pares tras el cruce de NAT; de lo contrario, utiliza un servidor de retransmisión DERP como vía de respaldo. Tailscale no reinventa el cifrado, sino que utiliza WireGuard como transporte.

**Plano de control mediante el servidor de coordinación.** Un servidor de coordinación central distribuye las claves públicas y un mapa de red que registra qué nodo posee qué direcciones y rutas. El servidor de coordinación ve los metadatos (quién puede comunicarse con quién y qué rutas están aprobadas), pero no el contenido de los paquetes de WireGuard. Cuando anuncia una ruta, el nodo lo comunica al plano de control; solo al aprobarse pasa a formar parte del mapa de red que reciben todos los nodos.

**En el nodo router.** Para que un nodo reenvíe tráfico para otros dispositivos, debe tener activado el reenvío IP y transportar los paquetes entre la interfaz de Tailscale y la red local. De forma predeterminada, Tailscale enmascara el tráfico reenviado mediante NAT de origen (SNAT): los dispositivos de destino de la red local ven como remitente la dirección local del nodo router, no la dirección de Tailscale del dispositivo que accede. Es el caso sencillo, porque así los paquetes de respuesta encuentran por sí solos el camino de vuelta al router. Si desactiva SNAT, los dispositivos de destino ven la dirección de origen real de Tailscale, pero entonces la red local debe saber cómo enrutar de vuelta el rango de Tailscale hacia el router.

**En el lado del cliente.** Un dispositivo solo utiliza rutas ajenas si las acepta. En los clientes gráficos de Windows y macOS, la aceptación de rutas de subred está preconfigurada; en Linux se activa con `--accept-routes`. Si el cliente acepta una ruta, la añade a su tabla de enrutamiento y la dirige a la interfaz de Tailscale. Los paquetes destinados a una dirección de ese rango se encapsulan entonces en WireGuard y se envían al nodo router. Con el Exit Node se aplica el mismo mecanismo, solo que en este caso la ruta predeterminada apunta al Exit Node, por lo que todo el tráfico pasa por él.

**La aprobación.** Que las rutas solo tengan efecto tras su aprobación es una característica de seguridad, no un rodeo: un nodo cualquiera no debe poder atraer sin más tráfico destinado a redes enteras. La aprobación puede hacerse manualmente en la administración o automáticamente mediante `autoApprovers` en las reglas de acceso (ACL). Los Exit Nodes y las rutas de subred se aprueban por separado.

## Límites

Ambas variantes tienen límites que influyen en la elección:

- **El nodo router es un cuello de botella y un punto único de fallo.** Todo el tráfico para la red anunciada pasa por este único nodo, por su cifrado de WireGuard y por su conexión. Para disponer de tolerancia a fallos, varios nodos pueden anunciar la misma ruta; Tailscale utiliza entonces uno de ellos y cambia en caso de fallo.
- **SNAT oculta el origen.** Con el NAT de origen preconfigurado, todos los accesos aparecen bajo la dirección del nodo router. Para el registro o las reglas de acceso en los dispositivos de destino que necesitan el origen real, debe desactivar SNAT y configurar la ruta de retorno en la red local.
- **Un Exit Node realmente redirige todo.** Todo el tráfico pasa por el nodo, con las consecuencias correspondientes para el rendimiento, la latencia y la confidencialidad. El operador del Exit Node ve el tráfico en el punto donde abandona la tailnet. Utilice como Exit Node solo nodos en los que confíe.
- **Las subredes superpuestas son un problema.** Si dos ubicaciones anuncian el mismo rango privado, por ejemplo `192.168.1.0/24`, un cliente no puede distinguirlas. Tailscale ofrece para ello una reescritura mediante IPv6 (`4via6`), que hace que los rangos sean inequívocos.
- **Las claves que caducan detienen el reenvío.** Si caduca la clave del nodo router, deja de ser accesible toda la red situada detrás de él. Para un nodo router permanente, desactive la caducidad de claves en la administración.

Para el acceso selectivo a servicios internos, el router de subred es casi siempre la elección correcta: abre únicamente lo necesario. Utilice el Exit Node cuando desee dirigir deliberadamente todo el tráfico saliente a través de una ubicación determinada.

## Historial de cambios

- **25 de octubre de 2026:** enlaces de Fuentes actualizados a las nuevas direcciones de Tailscale.

## Fuentes

1.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): anuncio de rutas, aprobación, comportamiento de SNAT y alta disponibilidad con varios routers.

2.  [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes): anuncio de la ruta predeterminada, selección en el cliente y acceso a la propia red local.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): interacción entre el plano de datos de WireGuard, el servidor de coordinación y los relés DERP.

4.  [WireGuard: visión general del protocolo](https://www.wireguard.com/protocol/): la base criptográfica del plano de datos que Tailscale utiliza como transporte.
