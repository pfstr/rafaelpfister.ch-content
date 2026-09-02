---
title: "RustDesk: configurar la alternativa de código abierto a TeamViewer"
navTitle: "Configurar RustDesk"
description: "RustDesk es un software de soporte remoto de código abierto bajo AGPL, gratuito y autoalojable. Cómo instalar el cliente en Windows (también de forma desatendida mediante MSI), cómo se establece la conexión a través del servidor público de intermediación, un servidor propio o una conexión directa, qué funciones necesita el soporte diario y dónde están los límites del uso gratuito."
date: "2026-09-01"
kategorie: "Soporte remoto y asistencia"
timeToRead: "9 min de lectura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-configurar-la-alternativa-de-codigo-abierto-a-teamviewer"
translationId: "article-425ae4b8d562ae41"
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/es/blog/rustdesk-configurar-la-alternativa-de-codigo-abierto-a-teamviewer
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:49:29.674Z
translationReview: automatic
---

# RustDesk: configurar la alternativa de código abierto a TeamViewer

TeamViewer y AnyDesk cubren el soporte remoto de forma fiable, pero requieren una licencia para el uso comercial y los precios aumentan con el número de dispositivos gestionados. RustDesk es una alternativa bajo la licencia AGPL-3.0: de código abierto, gratuita y sin obligación de licencia. El cliente funciona en Windows, macOS, Linux, Android e iOS, así como en el navegador. Está escrito en Rust y la interfaz en Flutter.

La diferencia decisiva frente a los productos comerciales reside en la intermediación: RustDesk separa el cliente de la infraestructura de servidor. Puede utilizar el servidor público de intermediación gratuito, operar su propio servidor o establecer una conexión directa sin ningún servidor de intermediación. Así, RustDesk puede utilizarse desde un equipo individual hasta una plataforma de soporte autoalojada, sin que los datos de conexión tengan que pasar por un proveedor.

## Los tres tipos de conexión

Antes de instalar, debería definir el tipo de conexión, ya que de ello dependen la configuración y los puertos abiertos.

| Tipo de conexión | Cómo funciona | Cuándo conviene |
|---|---|---|
| Servidor público de intermediación | Dos clientes se encuentran mediante el ID (número de nueve dígitos) en el servidor de RustDesk; la conexión se establece directamente o a través de un relay | Inicio rápido, pruebas, soporte ocasional privado |
| Servidor propio (self-hosted) | Opera usted mismo los componentes de servidor `hbbs` (intermediación) y `hbbr` (relay); todos los clientes introducen su dirección | Uso comercial, muchos dispositivos, control total de los datos |
| Conexión directa (Direct IP Access) | El cliente se conecta directamente a la dirección IP del equipo remoto sin servidor de intermediación | Ambos dispositivos son accesibles en la misma red o mediante una VPN |

El servidor público está destinado expresamente a pruebas y uso privado. Para un funcionamiento productivo y comercial, el proyecto recomienda un servidor propio, también porque el servicio público está limitado y no ofrece garantía de disponibilidad.

## Instalación en Windows

Descargue el instalador de la fuente oficial, los lanzamientos de GitHub del proyecto (`github.com/rustdesk/rustdesk`). Para Windows hay un archivo ejecutable y un paquete MSI. Para la instalación interactiva basta con hacer doble clic. Si desea desplegar RustDesk en varios ordenadores o en segundo plano, utilice el MSI con una instalación silenciosa:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/i <paket>` | Instala el paquete MSI especificado |
| `/qn` | Sin interfaz ni diálogos (silencioso) |
| `/norestart` | Evita un reinicio automático tras la instalación |

</details>

La instalación silenciosa configura el servicio `RustDesk`, que se ejecuta al iniciar el sistema y permite el acceso desatendido. Después de la instalación, puede obtener el ID del dispositivo mediante la línea de comandos sin abrir la interfaz:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

También puede establecer una contraseña fija para el acceso desatendido mediante la línea de comandos. Asigne una contraseña independiente y suficientemente larga, no la contraseña de inicio de sesión del usuario:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--get-id` | Muestra el ID de RustDesk de nueve dígitos del dispositivo |
| `--password <wert>` | Establece la contraseña fija para el acceso desatendido |
| `--silent-install` | Instala la versión ejecutable (`.exe`) como servicio sin interfaz |

</details>

## Introducir un servidor propio

Si opera su propio servidor de intermediación, introduzca en los clientes su dirección y la clave pública. En la interfaz, esto aparece en la configuración de red como servidor de ID, servidor relay y clave. Para la distribución masiva, la configuración también puede especificarse mediante un archivo o variables de entorno, de modo que cada cliente se inicie preconfigurado.

Un servidor propio requiere los dos componentes `hbbs` y `hbbr`, que normalmente se ejecutan como contenedores de Docker. Ambos requieren puertos abiertos para que los clientes puedan registrarse y utilizar un relay.

| Puerto | Protocolo | Componente y propósito |
|---|---|---|
| 21114 | TCP | Interfaz web de la versión Pro (solo allí) |
| 21115 | TCP | `hbbs`, prueba del tipo de NAT |
| 21116 | TCP y UDP | `hbbs`, registro (UDP) y establecimiento de conexión (TCP) |
| 21117 | TCP | `hbbr`, tráfico de relay |
| 21118, 21119 | TCP | Compatibilidad con clientes web |

Abra solo los puertos que realmente necesite su tipo de conexión y restrinja mediante el firewall el acceso a las redes desde las que se presta soporte.

## Conexión directa sin servidor de intermediación

Si ambos dispositivos son accesibles en la misma red o mediante una VPN, RustDesk funciona sin ningún servidor de intermediación. Para ello, active el acceso directo en el dispositivo de destino (en la interfaz, en seguridad, como "Activar acceso directo por IP"; internamente, el interruptor `direct-server`). El cliente escucha entonces en el puerto estándar 21118 (TCP). En la ventana de conexión, introduzca la dirección IP del equipo remoto en lugar del ID.

Limite el acceso directo mediante el firewall a la red desde la que accede. Si el acceso se realiza mediante una VPN, habilite el puerto solo para el rango de direcciones de la VPN, no para todo Internet.

## Funciones para el soporte diario

RustDesk cubre las funciones que necesita el soporte remoto en el día a día:

- Transmisión de pantalla y control remoto de teclado y ratón, con selección de monitor en caso de varias pantallas.
- Transferencia de archivos en ambas direcciones mediante una ventana dividida.
- Chat de texto durante la sesión.
- Acceso desatendido mediante una contraseña fija, para dispositivos sin un usuario presente.
- Grabación de sesiones como archivo de vídeo, automáticamente si se desea.
- Túneles TCP y reenvío para acceder localmente a servicios individuales del equipo remoto.
- Libreta de direcciones y varios dispositivos guardados, localmente en la versión gratuita y compartidos en el servidor en la versión Pro.

Para el soporte asistido, hay un aspecto importante: de forma predeterminada, RustDesk pregunta en el equipo remoto si se acepta la conexión e indica durante la sesión que hay un acceso en curso. Por tanto, la persona que está ante el dispositivo lo sabe. Solo una contraseña fija para el acceso desatendido elimina esta confirmación. Utilice el acceso desatendido únicamente en dispositivos cuyos usuarios sepan que el software está instalado y para qué sirve.

## Restricciones y límites

RustDesk sustituye a TeamViewer en muchos casos, pero tiene límites que debería conocer antes de utilizarlo:

- El servidor público de intermediación está limitado, no ofrece garantía de disponibilidad y no está previsto para un funcionamiento comercial continuado. Quien quiera trabajar de forma fiable debe autoalojarlo.
- Un servidor propio implica esfuerzo operativo: los contenedores, los puertos abiertos, los certificados y las actualizaciones son su responsabilidad.
- Una libreta de direcciones compartida en el servidor, la gestión centralizada de usuarios y la interfaz web de administración forman parte de la versión Pro, que es de pago a partir de un determinado número de dispositivos. El cliente y el funcionamiento básico siguen siendo gratuitos.
- Sin una contraseña fija no es posible el acceso desatendido, lo cual es correcto para el soporte asistido, pero impide el acceso espontáneo a un dispositivo desocupado.
- La amplitud funcional y la estabilidad de determinadas plataformas, especialmente en dispositivos móviles, no igualan a los productos comerciales en todos los detalles. Compruebe las funciones importantes para usted antes de cambiar.
- Algunos programas de seguridad detectan el software de soporte remoto como potencialmente no deseado. Si es necesario, añada una excepción y documente por qué está instalado el software.

Para el uso privado y el soporte de dispositivos individuales, basta la versión gratuita con el servidor público o una conexión directa. En cuanto gestione muchos dispositivos, trabaje comercialmente o necesite control total de los datos, requerirá un servidor propio, con el correspondiente esfuerzo operativo a cambio de independencia.

## Fuentes

1.  [RustDesk en GitHub](https://github.com/rustdesk/rustdesk): código fuente, lanzamientos con los instaladores y la licencia AGPL-3.0.

2.  [Documentación de RustDesk](https://rustdesk.com/docs/): instalación, servidor propio, puertos y configuración de los clientes.

3.  [rustdesk-server en GitHub](https://github.com/rustdesk/rustdesk-server): componentes de servidor `hbbs` y `hbbr`, incluida la vista general de puertos para el funcionamiento propio.
