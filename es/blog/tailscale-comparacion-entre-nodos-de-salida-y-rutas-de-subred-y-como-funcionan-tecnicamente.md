---
title: "Tailscale: comparación entre nodos de salida y rutas de subred, y cómo funcionan técnicamente"
navTitle: "Nodo de salida vs. subred"
description: "En Tailscale, los nodos de salida y los routers de subred son dos modos de funcionamiento relacionados, pero diferentes. Un router de subred habilita de forma selectiva determinados rangos de IP, mientras que un nodo de salida enruta todo el tráfico de Internet a través de sí mismo. Qué significa esto en la práctica, cómo lo implementa Tailscale mediante WireGuard, la aprobación de rutas y SNAT, y cuáles son los límites de cada variante."
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
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
translationOf: tailscale-exit-node-subnet-routes
url: https://rafaelpfister.ch/es/blog/tailscale-comparacion-entre-nodos-de-salida-y-rutas-de-subred-y-como-funcionan-tecnicamente
translationSourceHash: f05a193f13dd2b8aba3c9d049ea1c0a1fcc25b12c420a1d520f99854b7883a79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:01:04.801Z
translationReview: automatic
---

# Tailscale: comparación entre nodos de salida y rutas de subred, y cómo funcionan técnicamente

Un nodo de Tailscale inicialmente solo es accesible por sí mismo: a través de su dirección de Tailscale, y nada más. Para que un nodo dé a otros dispositivos acceso a más que a sí mismo, existen dos modos de funcionamiento que a menudo se confunden: el **router de subred** y el **nodo de salida**. Ambos amplían el alcance de un nodo, pero en direcciones distintas. Quien conoce la diferencia elige la variante adecuada y evita enrutar accidentalmente todo el tráfico a través de un ordenador ajeno.

La versión corta: un router de subred habilita **selectivamente determinados rangos de IP** detrás del nodo, por ejemplo, la red local con un NAS y una impresora. Un nodo de salida enruta **todo el tráfico de Internet** de un dispositivo a través de sí mismo, como una VPN clásica de túnel completo. Ambos se basan técnicamente en el mismo mecanismo: anunciar rutas. El nodo de salida es, en esencia, un caso especial del router de subred en el que se anuncia la ruta predeterminada.

## Router de subred: acceso selectivo a una red

Un router de subred anuncia uno o varios rangos de IP a los que puede acceder en la red local. Otros dispositivos de la red de Tailscale que aceptan estas rutas pueden acceder a través de él a los dispositivos del rango anunciado, aunque Tailscale no esté instalado en ellos. Esta es la forma de hacer accesible un NAS, una impresora o una interfaz de administración sin configurar un cliente VPN en cada dispositivo.

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
| `--advertise-exit-node` | Forma abreviada que anuncia `0.0.0.0/0` y `::/0`, ofreciendo así el nodo como nodo de salida |

</details>

El tráfico solo fluye después de que la ruta haya sido **aprobada** en la administración de Tailscale. Anunciarla por sí solo no basta; este es el error más frecuente: la ruta solo aparece en la tabla de enrutamiento de los dispositivos que la aceptan después de su aprobación.

## Nodo de salida: todo el tráfico a través de un nodo

Un nodo de salida anuncia la ruta predeterminada (`0.0.0.0/0` y `::/0`). Cuando un dispositivo selecciona este nodo de salida, **todo** su tráfico saliente de Internet pasa por el nodo, no solo el tráfico hacia una red específica. Esto resulta útil para acceder a Internet a través de una ubicación con una IP fija o para dirigir el tráfico a través de una salida de confianza en una red insegura.

La diferencia con la ruta de subred está en la selección en el lado del cliente: una ruta de subred se utiliza automáticamente en cuanto el dispositivo acepta la ruta y se dirige a un destino de ese rango. En cambio, un nodo de salida debe seleccionarse activamente, y entonces se aplica a todo el tráfico:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--exit-node=<IP oder Name>` | Selecciona un nodo de salida; vacío (`--exit-node=`) lo vuelve a desactivar |
| `--exit-node-allow-lan-access` | Permite el acceso a la propia red local incluso con un nodo de salida activo |

</details>

Precisamente por eso, en el día a día del soporte era incorrecto marcar el nodo de salida para acceder a un único NAS: habría redirigido todo el tráfico propio a través de un ordenador ajeno, en lugar de habilitar solo ese rango.

## Comparación

| Característica | Router de subred | Nodo de salida |
|---|---|---|
| Ruta anunciada | Rangos específicos, p. ej. `192.168.1.0/24` | Ruta predeterminada `0.0.0.0/0`, `::/0` |
| Uso por parte del cliente | Automático para destinos del rango | Debe seleccionarse activamente como nodo de salida |
| Alcance | Solo las redes anunciadas | Todo el tráfico de Internet |
| Aprobación en la administración | Por subred | Por separado como nodo de salida |
| Finalidad típica | Hacer accesibles servicios internos | Dirigir el tráfico saliente a través de una ubicación |

## Cómo lo implementa técnicamente Tailscale

Ambos modos de funcionamiento se basan en el mismo fundamento. Conviene separar los niveles.

**Plano de datos mediante WireGuard.** Cada nodo tiene un par de claves de WireGuard. El tráfico real entre dos nodos circula directamente como paquetes de WireGuard cifrados a través de UDP, cuando es posible de punto a punto tras la traversía de NAT; de lo contrario, a través de un servidor de retransmisión DERP como vía alternativa. Tailscale no reinventa el cifrado, sino que utiliza WireGuard como transporte.

**Plano de control mediante el servidor de coordinación.** Un servidor de coordinación central distribuye las claves públicas y un mapa de red que registra qué nodo posee qué direcciones y rutas. El servidor de coordinación ve los metadatos (quién puede hablar con quién y qué rutas están aprobadas), pero no el contenido de los paquetes de WireGuard. Cuando anuncia una ruta, el nodo lo comunica al plano de control; solo con la aprobación la ruta pasa a formar parte del mapa de red que reciben todos los nodos.

**En el nodo router.** Para que un nodo reenvíe tráfico para otros dispositivos, debe tener activado el reenvío de IP y transmitir los paquetes entre la interfaz de Tailscale y la red local. De forma predeterminada, Tailscale enmascara el tráfico reenviado con NAT de origen (SNAT): los dispositivos de destino de la red local ven como remitente la dirección local del nodo router, no la dirección de Tailscale del dispositivo que accede. Este es el caso sencillo, porque así los paquetes de respuesta encuentran automáticamente el camino de vuelta al router. Si desactiva SNAT, los dispositivos de destino ven la dirección de origen real de Tailscale, pero entonces la red local debe saber cómo enrutar de vuelta el rango de Tailscale hacia el router.

**En el lado del cliente.** Un dispositivo solo utiliza rutas ajenas si las acepta. En los clientes gráficos para Windows y macOS, la aceptación de rutas de subred viene activada de forma predeterminada; en Linux se activa con `--accept-routes`. Si el cliente acepta una ruta, la añade a su tabla de enrutamiento y la dirige a la interfaz de Tailscale. Los paquetes destinados a una dirección de ese rango se encapsulan entonces en WireGuard y se envían al nodo router. Con un nodo de salida se aplica el mismo mecanismo, solo que aquí la ruta predeterminada apunta al nodo de salida, por lo que todo el tráfico pasa por él.

**La aprobación.** Que las rutas solo surtan efecto tras su aprobación es una característica de seguridad, no un paso innecesario: un nodo cualquiera no debe poder atraer sin autorización tráfico de redes enteras. La aprobación puede hacerse manualmente en la administración o automáticamente mediante `autoApprovers` en las reglas de control de acceso (ACL). Los nodos de salida y las rutas de subred se aprueban por separado.

## Límites

Ambas variantes tienen límites que influyen en la elección:

- **El nodo router es un cuello de botella y un punto único de fallo.** Todo el tráfico de la red anunciada pasa por ese único nodo, por su cifrado WireGuard y por su conexión. Para disponer de tolerancia a fallos, varios nodos pueden anunciar la misma ruta; Tailscale utiliza entonces uno de ellos y cambia en caso de fallo.
- **SNAT oculta el origen.** Con el NAT de origen predeterminado, todos los accesos aparecen bajo la dirección del nodo router. Para el registro o las reglas de acceso en los dispositivos de destino que necesitan el origen real, debe desactivar SNAT y configurar la ruta de retorno en la red local.
- **Un nodo de salida realmente enruta todo.** Todo el tráfico pasa por el nodo, con las consecuencias correspondientes para el rendimiento, la latencia y la confidencialidad. El operador del nodo de salida ve el tráfico en el punto donde abandona la red de Tailscale. Utilice como nodo de salida únicamente nodos en los que confíe.
- **Las subredes superpuestas son un problema.** Si dos ubicaciones anuncian el mismo rango privado, por ejemplo `192.168.1.0/24`, un cliente no puede distinguirlas. Para ello, Tailscale ofrece una traducción mediante IPv6 (`4via6`), que hace que los rangos sean inequívocos.
- **Las claves caducadas detienen el reenvío.** Si caduca la clave del nodo router, deja de ser accesible toda la red situada detrás de él. Para un nodo router permanente, desactive la caducidad de claves en la administración.

Para el acceso selectivo a servicios internos, el router de subred casi siempre es la opción adecuada: solo habilita lo necesario. Utilice el nodo de salida cuando quiera dirigir deliberadamente todo el tráfico saliente a través de una ubicación determinada.

## Fuentes

1.  [Tailscale: routers de subred](https://tailscale.com/kb/1019/subnets): anuncio de rutas, aprobación, comportamiento de SNAT y alta disponibilidad con varios routers.

2.  [Tailscale: nodos de salida](https://tailscale.com/kb/1103/exit-nodes): anuncio de la ruta predeterminada, selección en el cliente y acceso a la propia red local.

3.  [Tailscale: cómo funciona Tailscale](https://tailscale.com/blog/how-tailscale-works): interacción entre el plano de datos de WireGuard, el servidor de coordinación y los relés DERP.

4.  [WireGuard: resumen del protocolo](https://www.wireguard.com/protocol/): la base criptográfica del plano de datos que Tailscale utiliza como transporte.
