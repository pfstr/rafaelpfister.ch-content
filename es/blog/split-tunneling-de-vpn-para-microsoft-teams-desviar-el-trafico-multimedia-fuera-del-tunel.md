---
title: "Split tunneling de VPN para Microsoft Teams: desviar el tráfico multimedia fuera del túnel"
navTitle: "Split tunneling de Teams"
description: "Las llamadas de Teams a través de una VPN sufren latencia, jitter y el desvío a través de la puerta de enlace VPN. El artículo muestra qué redes y puertos de Microsoft gestionan el tráfico multimedia, por qué el split tunneling basado en IP es superior a la exclusión por aplicación y cómo se implementa en VPN de consumo, WireGuard, OpenVPN y clientes empresariales."
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
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
translationOf: vpn-split-tunneling-microsoft-teams
url: https://rafaelpfister.ch/es/blog/split-tunneling-de-vpn-para-microsoft-teams-desviar-el-trafico-multimedia-fuera-del-tunel
translationSourceHash: 95e3cefa4946676022602866d6ef21ab92ef25ec8c5dd3ff4ab0219ba718a880
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:32:34.773Z
translationReview: automatic
---

Una llamada de Teams a través de una conexión VPN suele sonar peor que sin ella: la voz se entrecorta, el vídeo va a tirones y las transmisiones de pantalla tardan en cargarse. La causa suele ser el desvío que realiza el tráfico en tiempo real a través del túnel VPN, no Teams en sí. Por ello, Microsoft recomienda desde hace años dirigir el tráfico multimedia de Teams directamente a Internet mediante split tunneling, evitando la VPN. Este enfoque funciona con prácticamente cualquier producto VPN, desde clientes de consumo hasta puertas de enlace empresariales; la configuración solo difiere en los detalles.

## Por qué el tráfico en tiempo real sufre dentro del túnel

El audio y el vídeo de Teams funcionan mediante SRTP, un protocolo basado en UDP que depende de una latencia baja y poco jitter. Microsoft indica como valores objetivo menos de 100 ms de tiempo de ida y vuelta hasta la entrada de red de Microsoft más cercana y menos de 30 ms de jitter. Un túnel VPN empeora ambos valores de varias maneras.

En primer lugar, el túnel alarga la ruta: en vez de ir directamente al punto de entrada de Microsoft geográficamente más cercano, el tráfico pasa primero por la puerta de enlace VPN, que puede estar en el centro de datos del proveedor o de la empresa, y solo después llega a Microsoft. En segundo lugar, la capa de cifrado adicional consume tiempo de procesamiento y aumenta la sobrecarga por paquete; el flujo multimedia ya está cifrado con SRTP, y el cifrado VPN se añade como segunda capa. En tercer lugar, la puerta de enlace VPN es un cuello de botella compartido: en horas punta, todos los usuarios comparten su ancho de banda y sus búferes de paquetes, lo que genera precisamente el jitter al que el tráfico en tiempo real reacciona con mayor sensibilidad. En cuarto lugar, algunas configuraciones VPN bloquean UDP por completo o fuerzan TCP; entonces Teams recurre a TCP 443, lo que empeora aún más la calidad porque las retransmisiones de TCP no son adecuadas para medios en tiempo real.

Para el resto del tráfico de Teams (inicio de sesión, chat, acceso a archivos), todo ello apenas importa, ya que no es sensible al tiempo real. Por tanto, basta con excluir específicamente el tráfico multimedia.

## Las redes y los puertos relevantes

Microsoft publica todos los puntos de conexión de Microsoft 365 en formato legible por máquina y los divide en las categorías Optimize, Allow y Default. Para el split tunneling es relevante la categoría Optimize: comprende los pocos puntos de conexión críticos para la latencia con redes IP fijas, que en conjunto representan la mayor parte del volumen. Para los medios de Teams, se trata de los identificadores de punto de conexión 11 y 12 de la lista oficial:

| Red | Protocolo y puertos | Finalidad |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 a 3481, TCP 443 | Medios de Teams (audio, vídeo, transmisión de pantalla) |
| `52.112.0.0/14` | UDP 3478 a 3481, TCP 443 | Medios de Teams y relés de transporte |
| `52.122.0.0/15` | UDP 3478 a 3481, TCP 443 | Medios de Teams y relés de transporte |
| `2603:1063::/38` | UDP 3478 a 3481, TCP 443 | Los mismos servicios mediante IPv6 |

Los cuatro puertos UDP corresponden a las clases multimedia de audio (3478), vídeo (3479 y 3480) y transmisión de pantalla (3481); TCP 443 es la ruta de respaldo. Quien utilice IPv6 también debería excluir la red IPv6, pues de lo contrario una parte de las conexiones volverá a pasar por el túnel.

Estas redes son deliberadamente estables: Microsoft anuncia los cambios en los puntos de conexión Optimize a través del servicio web de puntos de conexión y mantiene la lista reducida, precisamente para que las empresas puedan incorporarlas en reglas de enrutamiento y firewall. Aun así, conviene comparar periódicamente con la lista oficial como parte de la rutina operativa.

## Basado en aplicaciones o en IP: dos enfoques con ventajas desiguales

Muchos clientes VPN ofrecen dos tipos de split tunneling: exclusiones por aplicación o exclusiones por IP de destino.

La exclusión por aplicación parece obvia, pero presenta dos debilidades con Teams. El nuevo Teams es una aplicación WebView2: el proceso principal se llama `ms-teams.exe`, pero parte del tráfico pasa por `msedgewebview2.exe`. Quien excluya solo el proceso principal no captará todo el tráfico; quien excluya también WebView2 desviará además del túnel el tráfico de otras aplicaciones WebView2 (como el nuevo Outlook). Y para Teams en el navegador, la exclusión por aplicación no sirve de nada, salvo que se excluya todo el navegador, con lo que todo el tráfico web evita la VPN.

En cambio, la exclusión basada en IP actúa a nivel de red y, por tanto, es independiente de si el tráfico procede de la aplicación Teams, de WebView2 o de una pestaña del navegador. Excluye exactamente lo que es crítico para la latencia y mantiene en el túnel el inicio de sesión, el chat y el resto del tráfico web. Por ello, para Teams el enfoque basado en IP es la mejor opción; la exclusión por aplicación sirve como complemento cuando se desea que todo el tráfico de Teams evite la VPN.

## Implementación en productos VPN habituales

El principio es el mismo en todas partes: las tres redes IPv4 (y, si es necesario, la red IPv6) se excluyen del túnel, de modo que las rutas del sistema operativo para estos destinos apunten a la interfaz física.

**VPN de consumo (Proton VPN, NordVPN, Surfshark y similares):** Los clientes para Windows y Android suelen ofrecer una opción de menú como «Split Tunneling» con una lista de exclusión para direcciones IP o subredes. Introduzca allí las tres redes en notación CIDR y restablezca la conexión VPN para que se apliquen las rutas. En macOS e iOS, la función falta en la mayoría de proveedores porque las API del sistema no permiten el split tunneling controlado por aplicaciones de esta forma.

**WireGuard:** WireGuard no conoce una lista de exclusión, sino únicamente la especificación `AllowedIPs`, que determina qué entra en el túnel. Las excepciones se crean sustituyendo `0.0.0.0/0` por la lista de todas las redes que no contienen el rango excluido. Nadie calcula esta lista complementaria a mano; calculadoras en línea como WireGuard AllowedIPs Calculator toman `0.0.0.0/0` como base, las tres redes de Microsoft como «Disallowed IPs» y proporcionan la línea terminada para el archivo de configuración.

**OpenVPN:** Con `redirect-gateway` activo, prevalecen las rutas más específicas. Tres líneas adicionales en la configuración del cliente dirigen las redes de Microsoft fuera del túnel:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` representa aquí la puerta de enlace predeterminada de la red local, no la puerta de enlace VPN.

**Clientes empresariales (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Aquí la empresa configura las excepciones de forma centralizada: en Cisco, como lista «Split Exclude» en la directiva de grupo; en GlobalProtect, como «Exclude Access Route». Microsoft documenta expresamente este procedimiento como el modelo recomendado para el tráfico de Microsoft 365 y proporciona la lista Optimize a través del servicio web de puntos de conexión, de modo que las excepciones puedan mantenerse actualizadas de forma automatizada. Quien, como empleado, esté detrás de una VPN corporativa no puede configurar la excepción por sí mismo, sino que debe solicitarla al equipo de red; el documento de Microsoft al respecto proporciona la base de argumentación adecuada.

**Herramientas integradas de Windows:** En una conexión VPN configurada con las herramientas integradas de Windows en modo dividido (`Set-VpnConnection -SplitTunneling $true`), solo entran en el túnel las redes añadidas mediante `Add-VpnConnectionRoute`. Mientras las redes de Microsoft no aparezcan allí, se enrutarán automáticamente de forma directa; entonces no es necesaria una exclusión explícita.

## Consideraciones de seguridad: qué evita el túnel

El split tunneling constituye una flexibilización deliberada del principio de dirigir todo el tráfico por el túnel. Antes de implementarlo, debe aclarar tres puntos.

Microsoft verá su propia dirección IP pública, pues esto es precisamente lo que se pretende: el flujo multimedia debe seguir la ruta más corta. Quien utilice una VPN principalmente para ocultar su ubicación pierde esta protección para las llamadas de Teams. El contenido no se ve afectado, porque SRTP cifra el flujo multimedia de extremo a extremo entre el cliente y la infraestructura de Microsoft.

En el entorno empresarial, la puerta de enlace de seguridad central pierde visibilidad sobre el tráfico excluido: la inspección TLS, las firmas IDS y el análisis de volumen dejan de aplicarse a estas redes. Dado que la excepción se limita a unas pocas redes asignadas de forma fija a Microsoft y a puertos definidos, Microsoft considera bajo este riesgo residual; los puntos de conexión Optimize están seleccionados precisamente para este fin. En cambio, una excepción general para aplicaciones completas o incluso para el navegador tiene una superficie de ataque considerablemente mayor y debe evitarse en entornos empresariales.

Por último, el Kill Switch: algunos clientes VPN aplican las excepciones de split tunneling solo después de restablecer la conexión o se comportan de forma distinta cuando el Kill Switch está activo. Por ello, tras cada modificación de la lista de exclusión debe restablecerse la conexión y realizarse una prueba de verificación.

## Comprobación: ¿el tráfico multimedia realmente va directo?

Puede comprobarse en dos niveles si la excepción funciona. A nivel de enrutamiento, PowerShell muestra qué interfaz elige Windows para un destino de las redes de Microsoft:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Si aparece la interfaz física (Ethernet o WLAN) en lugar del adaptador VPN, la ruta es correcta. A nivel de aplicación, Teams proporciona la confirmación: durante una llamada, la integridad de la llamada (en «Más acciones» en la ventana de llamada) muestra el tipo de conexión negociado, el tiempo de ida y vuelta y la tasa de pérdida de paquetes. Un tiempo de ida y vuelta que disminuye notablemente tras el cambio y el tipo de conexión UDP en lugar de TCP son los dos indicios de que la excepción funciona.

Si el tráfico sigue pasando por el túnel pese a que la ruta sea correcta, conviene revisar el orden de los adaptadores de red y las particularidades del cliente: algunos clientes VPN vuelven a imponer sus rutas con una métrica menor tras cada establecimiento de conexión, y una lista de exclusión obsoleta solo se hace evidente cuando Microsoft añade una red. Por ello, la comparación con la lista oficial de puntos de conexión debe seguir el mismo ritmo que otras comprobaciones periódicas de red.

## Fuentes

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): lista oficial de puntos de conexión; las redes multimedia de Teams figuran con los ID 11 y 12 en la categoría Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guía de implementación de Microsoft para VPN empresariales, incluida la justificación de la evaluación de riesgos.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): los principios detrás de la salida local a Internet, incluidos los valores objetivo de latencia para medios en tiempo real.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): ejemplo de un cliente de consumo con split tunneling basado en IP y aplicaciones en Windows y Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calculadora para la lista complementaria cuando las excepciones deben implementarse mediante AllowedIPs.
