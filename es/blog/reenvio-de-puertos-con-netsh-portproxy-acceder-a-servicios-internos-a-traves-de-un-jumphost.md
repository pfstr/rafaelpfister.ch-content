---
title: "Reenvío de puertos con netsh portproxy: acceder a servicios internos a través de un jumphost"
navTitle: "netsh portproxy"
description: "Windows incluye un reenvío de puertos TCP integrado con netsh interface portproxy. Junto con una VPN como Tailscale, permite acceder desde el exterior a un servicio interno, por ejemplo la interfaz de un NAS, sin exponerlo públicamente. Cómo configurar, proteger y eliminar el reenvío, y cuáles son sus límites: sin UDP, sin cifrado adicional y con posibles inconvenientes relacionados con certificados y redirecciones."
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
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Jumphost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
translationOf: windows-portproxy-portweiterleitung
url: https://rafaelpfister.ch/es/blog/reenvio-de-puertos-con-netsh-portproxy-acceder-a-servicios-internos-a-traves-de-un-jumphost
translationSourceHash: a4888a85b953fbf7b2248232b7b7361e752300872cdb570d6fd15b1cb806ef89
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:03:49.021Z
translationReview: automatic
---

# Reenvío de puertos con netsh portproxy: acceder a servicios internos a través de un jumphost

Un servicio interno suele escuchar únicamente en la red local: la interfaz web de un NAS, el panel de una impresora o una página de administración. Si desea acceder a él desde el exterior sin exponer el servicio a Internet, necesita una vía a través de un equipo que vea ambos lados. Windows incluye para ello una herramienta integrada: `netsh interface portproxy` reenvía conexiones TCP entrantes a otro destino. En combinación con una VPN como Tailscale o WireGuard, un equipo de la red de destino se convierte en un jumphost a través del cual puede acceder al servicio interno.

Un ejemplo concreto: un NAS con la interfaz web en `10.0.0.245:5000` solo es accesible en la red local. En la misma red hay un PC con Windows que también es accesible mediante VPN. Configure en este PC un reenvío de puertos desde su dirección VPN hacia el NAS y, a continuación, abra la interfaz del NAS en el navegador mediante la dirección VPN del PC. El servicio permanece en la red interna; solo el jumphost es accesible mediante la VPN.

## Cómo funciona portproxy

`portproxy` es un componente del servicio IP Helper (`iphlpsvc`). El servicio acepta conexiones en un puerto local y las reenvía a un destino. Es un relay TCP puro de capa de aplicación: no una regla NAT de firewall, sino un proceso que copia bytes entre dos conexiones. Si `iphlpsvc` no se está ejecutando, ningún reenvío funciona. El servicio está disponible de forma predeterminada; su tipo de inicio debe estar configurado como automático si el reenvío debe sobrevivir a un reinicio.

## Configuración

Un reenvío requiere dos pasos: la regla de portproxy y una regla de firewall que permita el acceso al listener. Ejecute ambos en una consola o PowerShell con derechos de administrador.

Primero, el reenvío. Se enlaza a una dirección local y un puerto, y apunta a una IP y un puerto de destino:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `v4tov4` | IPv4 escucha y IPv4 conecta; también son posibles: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Dirección local en la que se escucha; aquí, la dirección VPN del jumphost, para que solo se acepte tráfico a través de la VPN |
| `listenport` | Puerto local en el que se escucha |
| `connectaddress` | IP de destino a la que se reenvía (el servicio interno) |
| `connectport` | Puerto de destino del servicio interno |

</details>

El enlace a la dirección VPN en lugar de a `0.0.0.0` es la primera medida de seguridad: el listener solo aparece en la interfaz VPN, no en todas las tarjetas de red del jumphost. La segunda medida de seguridad es el firewall. Abra el puerto del listener exclusivamente para el rango de direcciones de su VPN, no para todas las direcciones:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Direction Inbound` | Regla para tráfico entrante |
| `-Protocol TCP` | portproxy solo reenvía TCP; por eso, TCP |
| `-LocalPort 5000` | El puerto del listener de la regla de portproxy |
| `-RemoteAddress 100.64.0.0/10` | Solo se permiten fuentes de este rango; aquí, el rango de Tailscale; de lo contrario, el bloque CIDR de su VPN |

</details>

## Comprobar y usar

Primero compruebe en el propio jumphost si el servicio interno es accesible y muestre el reenvío activo:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Si el destino responde y la regla aparece en la lista, pruébelo desde su dispositivo remoto. Ahora puede acceder al servicio mediante la dirección y el puerto del jumphost:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

En el navegador, abra entonces `http://100.100.10.10:5000`. Si necesita varios puertos del mismo servicio, por ejemplo 5000 y 5001 para http y https, cree una regla de portproxy independiente y la correspondiente apertura de firewall para cada puerto.

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
| `reset` | Eliminar todas las reglas de portproxy |

</details>

Las reglas se almacenan en el Registro bajo `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` y sobreviven a un reinicio. Solo son visibles mediante `netsh` o directamente en el Registro, no en la interfaz gráfica del firewall.

## Alternativas

`portproxy` resulta práctico si el jumphost ya usa Windows y no desea instalar nada adicional. Dos alternativas resuelven el mismo problema con otras características.

Un túnel SSH con reenvío local (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) cifra el trayecto hasta el propio jumphost y funciona en distintas plataformas. Requiere un servidor SSH en el jumphost y solo existe mientras la sesión SSH permanezca activa.

Un router de subred de Tailscale (`tailscale up --advertise-routes=10.0.0.0/24`) hace que toda la subred interna sea accesible para sus dispositivos VPN. De este modo, puede dirigirse al servicio interno directamente mediante su IP real, sin reenvío por puerto. Es la vía más directa si desea acceder a varios dispositivos internos, pero requiere aprobar la ruta en la administración de Tailscale.

## Límites

Un reenvío de puertos con portproxy resuelve el acceso, pero tiene límites claros que debe conocer antes de utilizarlo:

- **Solo TCP.** `portproxy` reenvía exclusivamente TCP. Los servicios que necesitan UDP (DNS, muchas VPN y protocolos de juego, algunas transmisiones de vídeo) no pueden implementarse de esta forma.
- **Sin cifrado adicional.** El reenvío copia los bytes sin modificarlos. La confidencialidad del trayecto la proporciona exclusivamente la VPN mediante la cual accede al jumphost. En una red de transporte sin cifrar, el tráfico quedaría desprotegido.
- **Advertencia de certificado con HTTPS mediante IP.** Si reenvía un servicio HTTPS y accede a él mediante la IP del jumphost, el certificado del destino no coincide con la dirección solicitada. El navegador muestra una advertencia. Puede aceptarse para una prueba breve, pero no para uso permanente.
- **Redirecciones y direcciones absolutas.** Algunas interfaces web redirigen por sí mismas a su nombre de host o a otro puerto, o crean enlaces absolutos con su dirección interna. En ese caso, el acceso a través del jumphost falla aunque el reenvío esté configurado. Estos servicios necesitan un proxy inverso real en lugar de un simple relay de puertos.
- **Enlace a una dirección que debe existir al iniciar.** Si la regla se enlaza a una `listenaddress` concreta, dicha dirección debe estar disponible al iniciar el servicio. Si la interfaz VPN se activa más tarde, el enlace puede fallar hasta que se reinicie el servicio o se vuelva a establecer la regla.
- **Una vía adicional hacia la red interna.** Cada reenvío es una ruta desde el exterior hacia un servicio interno. Restrinja estrictamente el firewall al rango VPN, enlácelo a la dirección VPN y elimine el reenvío en cuanto deje de necesitarlo.

## Eliminar de nuevo

Tras finalizar el trabajo, elimine el reenvío y la regla de firewall:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Un reenvío de puertos es una herramienta para acceso específico y limitado en el tiempo, no para un canal permanentemente abierto. Para operar de forma permanente un servicio interno a través de Internet, un proxy inverso con un certificado válido o una VPN con enrutamiento de subredes es una solución más adecuada.

## Fuentes

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): referencia de los subcomandos, las variantes de protocolo y la dependencia del servicio IP Helper.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): parámetros de la regla de firewall, incluida la restricción a rangos de direcciones mediante RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): hacer accesible toda una subred a través de la VPN, como alternativa al reenvío por puerto.
