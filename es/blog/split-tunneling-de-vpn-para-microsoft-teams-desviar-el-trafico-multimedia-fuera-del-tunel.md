---
title: "Split tunneling de VPN para Microsoft Teams: dirigir el tráfico multimedia fuera del túnel"
navTitle: "Split Tunneling de Teams"
description: "Las llamadas de Teams a través de una VPN sufren latencia, jitter y el desvío a través de la puerta de enlace VPN. El artículo muestra qué redes y puertos de Microsoft gestionan el tráfico multimedia, por qué el split tunneling basado en IP es superior a la exclusión por aplicación y cómo implementarlo en VPN de consumo, WireGuard, OpenVPN y clientes empresariales."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min de lectura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "split-tunneling-de-vpn-para-microsoft-teams-desviar-el-trafico-multimedia-fuera-del-tunel"
translationId: "article-d15f1e7ff6af231c"
translationOf: vpn-split-tunneling-microsoft-teams
translationSourceHash: 9a96126164a12f1f411065507313733f451161a25c0cd051778cc306fd32ed4c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:26:21.024Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/split-tunneling-de-vpn-para-microsoft-teams-desviar-el-trafico-multimedia-fuera-del-tunel
---

Una llamada de Teams a través de una conexión VPN suele sonar peor que sin ella: la voz se entrecorta, el vídeo se ralentiza y las presentaciones de pantalla se cargan con retraso. La causa suele ser el desvío que el tráfico en tiempo real realiza a través del túnel VPN, no Teams en sí. Por eso, Microsoft recomienda desde hace años dirigir el tráfico multimedia de Teams directamente a Internet mediante split tunneling, evitando la VPN. Este enfoque funciona con prácticamente cualquier producto VPN, desde clientes de consumo hasta gateways empresariales; la configuración solo difiere en los detalles.

## Por qué el tráfico en tiempo real sufre en el túnel

El audio y el vídeo de Teams funcionan mediante SRTP, un protocolo basado en UDP que depende de una baja latencia y poco jitter. Microsoft establece como valores objetivo menos de 100 ms de tiempo de ida y vuelta hasta el punto de entrada de red de Microsoft más cercano y menos de 30 ms de jitter. Un túnel VPN empeora ambos valores de varias maneras.

En primer lugar, el túnel alarga el recorrido: en vez de dirigirse directamente al punto de entrada de Microsoft geográficamente más próximo, el tráfico va primero a la puerta de enlace VPN, que puede estar en el centro de datos del proveedor o de la empresa, y solo entonces a Microsoft. En segundo lugar, la capa adicional de cifrado consume tiempo de procesamiento y aumenta la sobrecarga por paquete; el flujo multimedia ya está cifrado con SRTP y el cifrado de la VPN se añade como una segunda capa. En tercer lugar, la puerta de enlace VPN es un cuello de botella compartido: en horas punta, todos los usuarios comparten su ancho de banda y sus búferes de paquetes, lo que genera precisamente el jitter al que el tráfico en tiempo real reacciona con mayor sensibilidad. En cuarto lugar, algunas configuraciones VPN bloquean UDP por completo o fuerzan TCP; Teams recurre entonces a TCP 443, lo que vuelve a empeorar la calidad, ya que las retransmisiones de TCP no son adecuadas para medios en tiempo real.

Para el resto del tráfico de Teams (inicio de sesión, chat, acceso a archivos), todo ello apenas importa, ya que no es sensible al tiempo real. Por tanto, basta con excluir específicamente el tráfico multimedia.

## Las redes y los puertos relevantes

Microsoft publica todos los puntos de conexión de Microsoft 365 en formato legible por máquina y los divide en las categorías Optimize, Allow y Default. Para el split tunneling es relevante la categoría Optimize: incluye los pocos puntos de conexión críticos para la latencia con redes IP fijas, que en conjunto representan la mayor parte del volumen. Para los medios de Teams, se trata del ID de punto de conexión 11 de la lista oficial. El ID 12 incluye las mismas redes IPv4 y otras redes IPv6 para TCP 443 y 80, así como UDP 443, pero pertenece a la categoría Allow. La red que también se indicaba anteriormente `13.107.64.0/18` fue eliminada por Microsoft de ambas entradas en 2024 (versión 2024053000 para el ID 12, 2024073100 para el ID 11):

| Red | Protocolo y puertos | Finalidad |
|---|---|---|
| `52.112.0.0/14` | UDP 3478 a 3481 (ID 11), TCP 443 (ID 12) | Medios de Teams y relés de transporte |
| `52.122.0.0/15` | UDP 3478 a 3481 (ID 11), TCP 443 (ID 12) | Medios de Teams y relés de transporte |
| `2603:1063::/38` | UDP 3478 a 3481 (ID 11), TCP 443 (ID 12) | Los mismos servicios mediante IPv6 |

Los cuatro puertos UDP corresponden a las clases multimedia de audio (3478), vídeo (3479 y 3480) y presentación de pantalla (3481); TCP 443 es la vía de respaldo. Quien utilice IPv6 también debe excluir la red IPv6; de lo contrario, parte de las conexiones volverá a pasar por el túnel.

Estas redes son deliberadamente estables: Microsoft anuncia los cambios en los puntos de conexión Optimize a través del servicio web de endpoints y mantiene la lista reducida, precisamente para que las empresas puedan incorporarlas a reglas de enrutamiento y firewall. Aun así, una comparación ocasional con la lista oficial debe formar parte de la rutina operativa.

## Basado en aplicaciones o en IP: dos enfoques con ventajas desiguales

Muchos clientes VPN ofrecen dos tipos de split tunneling: excepciones por aplicación o excepciones por IP de destino.

La exclusión por aplicación parece obvia, pero presenta dos debilidades con Teams. El nuevo Teams es una aplicación WebView2: el proceso principal se llama `ms-teams.exe`, pero parte del tráfico pasa por `msedgewebview2.exe`. Quien excluya solo el proceso principal no abarcará todo el tráfico; quien excluya también WebView2 dirigirá fuera del túnel el tráfico de otras aplicaciones WebView2, como el nuevo Outlook. Además, para Teams en el navegador la exclusión por aplicación no sirve de nada, salvo que se excluya todo el navegador, con lo que todo el tráfico web evita la VPN.

En cambio, la exclusión basada en IP actúa a nivel de red y, por tanto, no depende de si el tráfico procede de la aplicación Teams, de WebView2 o de una pestaña del navegador. Excluye exactamente lo que es crítico para la latencia y mantiene en el túnel el inicio de sesión, el chat y el resto del tráfico web. Por ello, para Teams el enfoque basado en IP es la mejor opción; la exclusión por aplicación sirve como complemento cuando realmente se desea que todo el tráfico de Teams evite la VPN.

## Implementación en productos VPN habituales

El principio es siempre el mismo: las dos redes IPv4, y la red IPv6 si es necesario, se excluyen del túnel para que las rutas del sistema operativo hacia esos destinos apunten a la interfaz física.

**VPN de consumo (Proton VPN, NordVPN, Surfshark y similares):** Los clientes para Windows y Android suelen ofrecer una opción de menú como «Split Tunneling» con una lista de exclusiones para direcciones IP o subredes. Introduzca allí las redes en notación CIDR y restablezca la conexión VPN para que se apliquen las rutas. En macOS e iOS, la función no está disponible en la mayoría de los proveedores porque las API del sistema no permiten este tipo de split tunneling controlado por aplicaciones.

**WireGuard:** WireGuard no conoce una lista de exclusiones, sino únicamente la especificación `AllowedIPs` que determina qué entra en el túnel. Las excepciones se crean sustituyendo `0.0.0.0/0` por la lista de todas las redes que no contienen el rango excluido. Nadie calcula manualmente esta lista complementaria; calculadoras en línea como WireGuard AllowedIPs Calculator toman `0.0.0.0/0` como base, las redes de Microsoft como «Disallowed IPs» y proporcionan la línea terminada para el archivo de configuración.

**OpenVPN:** Con `redirect-gateway` activo, las rutas más específicas tienen prioridad. Dos líneas adicionales en la configuración del cliente dirigen las redes de Microsoft fuera del túnel:

```text
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

En este caso, `net_gateway` representa la puerta de enlace predeterminada de la red local, no la puerta de enlace VPN.

**Clientes empresariales (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Aquí la empresa configura las excepciones de forma centralizada: en Cisco, como lista «Split Exclude» en la política de grupo; en GlobalProtect, como «Exclude Access Route». Microsoft documenta expresamente este procedimiento como el modelo recomendado para el tráfico de Microsoft 365 y proporciona la lista Optimize mediante el servicio web de endpoints, de modo que las excepciones puedan mantenerse actualizadas automáticamente. Quien, como empleado, esté detrás de una VPN corporativa no puede configurar la excepción por sí mismo, sino que debe solicitarla al equipo de red; el documento de Microsoft al respecto es la base adecuada para argumentarlo.

**Herramientas integradas de Windows:** En una conexión VPN configurada con las herramientas integradas de Windows en modo dividido (`Set-VpnConnection -SplitTunneling $true`), solo las redes registradas mediante `Add-VpnConnectionRoute` entran en el túnel. Mientras las redes de Microsoft no aparezcan allí, pasarán automáticamente de forma directa; en ese caso no es necesaria una exclusión explícita.

## Consideraciones de seguridad: qué pasa fuera del túnel

El split tunneling es una flexibilización deliberada del principio de dirigir todo el tráfico a través del túnel. Antes de implementarlo, deben aclararse tres puntos.

La dirección IP pública propia será visible para Microsoft, pues precisamente eso es lo que se pretende: el flujo multimedia debe seguir la ruta más corta. Quien utilice una VPN principalmente para ocultar su ubicación renuncia a esta protección para las llamadas de Teams. El contenido no se ve afectado, ya que SRTP cifra el flujo multimedia de extremo a extremo entre el cliente y la infraestructura de Microsoft.

En entornos corporativos, la puerta de enlace central de seguridad pierde visibilidad sobre el tráfico excluido: la inspección TLS, las firmas IDS y el análisis de volumen dejan de aplicarse a estas redes. Dado que la excepción se limita a unas pocas redes asignadas de forma fija a Microsoft y a puertos definidos, Microsoft considera bajo este riesgo residual; los puntos de conexión Optimize están seleccionados precisamente para ello. En cambio, una excepción general para aplicaciones completas o incluso para el navegador tiene una superficie de ataque considerablemente mayor y debe evitarse en entornos corporativos.

Y, por último, el Kill Switch: algunos clientes VPN solo aplican las excepciones de split tunneling después de restablecer la conexión o se comportan de forma distinta con el Kill Switch activado. Por tanto, tras cada cambio en la lista de exclusiones se debe restablecer la conexión y realizar una prueba de comprobación.

## Comprobación: ¿el tráfico multimedia realmente va directo?

Puede comprobarse si la excepción funciona en dos niveles. A nivel de enrutamiento, PowerShell muestra qué interfaz elige Windows para un destino en las redes de Microsoft:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Si aparece la interfaz física, Ethernet o WLAN, en lugar del adaptador VPN, la ruta es correcta. A nivel de aplicación, Teams mismo proporciona la confirmación: durante una llamada, la integridad de la llamada, en «Más acciones» en la ventana de llamada, muestra el tipo de conexión negociado, el tiempo de ida y vuelta y la tasa de pérdida de paquetes. Un tiempo de ida y vuelta que disminuye notablemente tras el cambio y el tipo de conexión UDP en lugar de TCP son las dos señales de que la excepción funciona.

Si el tráfico continúa pasando por el túnel a pesar de que la ruta sea correcta, conviene revisar el orden de los adaptadores de red y las particularidades del cliente: algunos clientes VPN fuerzan de nuevo sus rutas con una métrica inferior tras cada establecimiento de conexión, y una lista de exclusiones obsoleta solo se hace evidente cuando Microsoft añade una red. Por ello, la comparación con la lista oficial de endpoints debe seguir el mismo ritmo que otras comprobaciones periódicas de red.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: Microsoft eliminó la red 13.107.64.0/18 de la lista de endpoints en 2024; se ha retirado de la tabla y del ejemplo de OpenVPN. También se ha corregido: solo el ID 11 pertenece a la categoría Optimize; el ID 12 pertenece a la categoría Allow.

## Fuentes

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): lista oficial de endpoints; las redes multimedia de Teams figuran bajo el ID 11 en la categoría Optimize, y el ID 12 contiene las mismas redes en la categoría Allow.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guía de implementación de Microsoft para VPN empresariales, incluida la justificación de la evaluación de riesgos y un ejemplo con las redes multimedia de Teams `52.112.0.0/14`, `52.122.0.0/15` y `2603:1063::/38`.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): los principios detrás de la salida local a Internet, incluidos los valores objetivo de latencia para medios en tiempo real.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): ejemplo de un cliente de consumo con split tunneling basado en IP y en aplicaciones en Windows y Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calculadora para la lista complementaria cuando las excepciones deben configurarse mediante AllowedIPs.

6.  [Microsoft 365 IP and URL web service: cambios Worldwide](https://endpoints.office.com/changes/worldwide/2024010100?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7): registro de cambios de la lista de endpoints, incluida la eliminación de `13.107.64.0/18` de los ID 12 (versión 2024053000) y 11 (versión 2024073100).
