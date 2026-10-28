---
title: "Reenvío de puertos con netsh portproxy: acceder a servicios internos a través de un host de salto"
navTitle: "netsh portproxy"
description: "Windows incluye un reenvío de puertos TCP integrado con netsh interface portproxy. Junto con una VPN como Tailscale, permite acceder desde el exterior a un servicio interno, por ejemplo la interfaz de un NAS, sin exponerlo públicamente. Cómo configurar, proteger y eliminar el reenvío, y cuáles son sus límites: sin UDP, sin cifrado adicional y con posibles problemas de certificados y redirecciones."
date: "2026-09-02"
kategorie: "Windows y redes"
timeToRead: "9 min de lectura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "reenvio-de-puertos-con-netsh-portproxy-acceder-a-servicios-internos-a-traves-de-un-jumphost"
translationId: "article-236adcb4ae982572"
translationOf: windows-portproxy-portweiterleitung
translationSourceHash: 152d2f6d20774179128718ab4e218fcc0cd76cbcf0706060df7d60f30e012900
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:55:35.125Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/reenvio-de-puertos-con-netsh-portproxy-acceder-a-servicios-internos-a-traves-de-un-jumphost
---

# Reenvío de puertos con netsh portproxy: acceder a servicios internos a través de un host de salto

Un servicio interno suele escuchar únicamente en la red local: la interfaz web de un NAS, el panel de una impresora o una página de administración. Si desea acceder a él desde el exterior sin ponerlo en Internet, necesita una ruta a través de un equipo que vea ambos lados. Windows incluye una herramienta integrada para ello: `netsh interface portproxy` reenvía conexiones TCP entrantes a otro destino. En combinación con una VPN como Tailscale o WireGuard, un equipo de la red de destino se convierte en el host de salto mediante el que puede acceder al servicio interno.

Un ejemplo concreto: un NAS cuya interfaz web está en `10.0.0.245:5000` solo es accesible desde la red local. En la misma red hay un PC con Windows que además es accesible mediante VPN. Si configura en este PC un reenvío de puertos desde su dirección VPN hacia el NAS, podrá abrir después la interfaz del NAS en el navegador mediante la dirección VPN del PC. El servicio permanece en la red interna; solo se puede acceder al host de salto a través de la VPN.

## Cómo funciona portproxy

`portproxy` forma parte del servicio IP Helper (`iphlpsvc`). El servicio acepta conexiones en un puerto local y las retransmite a un destino. Es un relé TCP puro en la capa de aplicación: no es una regla NAT de firewall, sino un proceso que copia bytes entre dos conexiones. Si `iphlpsvc` no está en ejecución, ningún reenvío funcionará. El servicio está disponible de forma predeterminada; su tipo de inicio debería estar configurado como automático si el reenvío debe sobrevivir a un reinicio.

## Configuración

Un reenvío requiere dos pasos: la regla de portproxy y una regla de firewall que permita el acceso al listener. Ejecute ambos en un símbolo del sistema o PowerShell con privilegios de administrador.

Primero, el reenvío. Se vincula a una dirección y un puerto locales y apunta a una IP y un puerto de destino:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `v4tov4` | IPv4 escucha, IPv4 conecta; también es posible: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Dirección local en la que se escucha; aquí, la dirección VPN del host de salto, para que las conexiones entren solo mediante la VPN |
| `listenport` | Puerto local en el que se escucha |
| `connectaddress` | IP de destino a la que se retransmite el tráfico (el servicio interno) |
| `connectport` | Puerto de destino del servicio interno |

</details>

La vinculación a la dirección VPN en lugar de `0.0.0.0` es la primera medida de protección: el listener aparece únicamente en la interfaz VPN, no en todas las tarjetas de red del host de salto. La segunda medida es el firewall. Abra el puerto del listener exclusivamente para el rango de direcciones de su VPN, no para todas las direcciones:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Direction Inbound` | Regla para tráfico entrante |
| `-Protocol TCP` | portproxy solo reenvía TCP, por eso TCP |
| `-LocalPort 5000` | El puerto del listener de la regla portproxy |
| `-RemoteAddress 100.64.0.0/10` | Solo se permiten orígenes de este rango; aquí, el rango de Tailscale; en otro caso, el bloque CIDR de su VPN |

</details>

## Comprobar y utilizar

Primero compruebe en el propio host de salto si el servicio interno es accesible y muestre el reenvío activo:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Si el destino responde y la regla aparece en la lista, pruébelo desde su dispositivo remoto. Ahora se puede acceder al servicio mediante la dirección y el puerto del host de salto:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

A continuación, abra `http://100.100.10.10:5000` en el navegador. Si necesita varios puertos del mismo servicio, por ejemplo 5000 y 5001 para http y https, cree una regla portproxy independiente y la correspondiente autorización de firewall para cada puerto.

## Resumen tipo página de manual

Los subcomandos más importantes de `netsh interface portproxy`:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Comando | Finalidad |
|---|---|
| `add v4tov4 …` | Crear un reenvío (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Mostrar los reenvíos IPv4 activos |
| `show all` | Mostrar todos los reenvíos de todas las variantes de protocolo |
| `delete v4tov4 listenaddress=… listenport=…` | Eliminar un reenvío |
| `reset` | Eliminar todas las reglas portproxy |

</details>

Las reglas se almacenan en el Registro, en `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy`, y sobreviven a un reinicio. Solo son visibles mediante `netsh` o directamente en el Registro, no en la interfaz gráfica del firewall.

## Alternativas

`portproxy` resulta práctico si el host de salto ya utiliza Windows y no desea instalar nada más. Dos alternativas resuelven el mismo problema con otras características.

Un túnel SSH con reenvío local (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) cifra el trayecto hasta el propio host de salto y funciona en varias plataformas. Requiere un servidor SSH en el host de salto y solo existe mientras la sesión SSH esté activa.

Un router de subred Tailscale (`tailscale set --advertise-routes=10.0.0.0/24`) permite que todos sus dispositivos VPN accedan a toda la subred interna. Entonces puede dirigirse directamente al servicio interno con su IP real, sin un reenvío por puerto. Es la vía más directa si desea acceder a varios dispositivos internos, pero requiere aprobar la ruta en la administración de Tailscale.

## Límites

Un reenvío de puertos con portproxy resuelve el acceso, pero tiene límites claros que debería conocer antes de utilizarlo:

- **Solo TCP.** `portproxy` reenvía exclusivamente TCP. No permite reproducir servicios que necesitan UDP (DNS, muchos protocolos de VPN y juegos, y algunas transmisiones de vídeo).
- **Sin cifrado adicional.** El reenvío copia los bytes sin modificarlos. La confidencialidad del trayecto la proporciona únicamente la VPN mediante la que accede al host de salto. En una red de transporte sin cifrar, el tráfico quedaría desprotegido.
- **Advertencia de certificado al usar HTTPS mediante IP.** Si reenvía un servicio HTTPS y lo abre mediante la IP del host de salto, el certificado del destino no coincide con la dirección solicitada. El navegador muestra una advertencia. Puede aceptarse para una prueba breve, pero no para un uso permanente.
- **Redirecciones y direcciones absolutas.** Algunas interfaces web redirigen por sí mismas a su nombre de host o a otro puerto, o construyen enlaces absolutos con su dirección interna. Entonces, el acceso mediante el host de salto falla aunque el reenvío esté configurado. Estos servicios requieren un proxy inverso real en lugar de un simple relé de puertos.
- **Vinculación a una dirección que debe existir al inicio.** Si la regla se vincula a una `listenaddress` concreta, esa dirección debe estar disponible al iniciar el servicio. Si la interfaz VPN se activa más tarde, la vinculación puede fallar hasta que se reinicie el servicio o se vuelva a configurar la regla.
- **Una vía adicional hacia la red interna.** Cada reenvío es una ruta desde el exterior hacia un servicio interno. Restrinja estrictamente el firewall al rango VPN, vincule a la dirección VPN y elimine el reenvío en cuanto deje de necesitarlo.

## Eliminar de nuevo

Tras terminar, elimine el reenvío y la regla de firewall:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Un reenvío de puertos es una herramienta para un acceso específico y limitado en el tiempo, no para un canal permanentemente abierto. Para operar permanentemente un servicio interno a través de Internet, un proxy inverso con certificado válido o una VPN con enrutamiento de subred es la solución más adecuada.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: las rutas de subred se anuncian con tailscale set --advertise-routes en lugar de tailscale up. Se actualizaron los enlaces de Fuentes a las nuevas direcciones de Microsoft Learn y Tailscale.

## Fuentes

1.  [netsh interface (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface): Referencia de comandos de `netsh interface` con la sintaxis de los subcomandos portproxy y las variantes de protocolo.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps): Parámetros de la regla de firewall, incluida la restricción a rangos de direcciones mediante RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Permite acceder a toda una subred mediante la VPN, como alternativa al reenvío por puerto; las rutas se anuncian con `tailscale set --advertise-routes`.
