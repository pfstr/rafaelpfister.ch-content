---
title: "RustDesk: configurar la alternativa de código abierto a TeamViewer"
navTitle: "Configurar RustDesk"
description: "RustDesk es un software de acceso remoto de código abierto bajo AGPL, gratuito y autoalojable. Cómo instalar el cliente en Windows (también de forma desatendida mediante MSI), cómo se establece la conexión a través del servidor de intermediación público, de un servidor propio o de una conexión directa, qué funciones se necesitan en el día a día del soporte y dónde están los límites del uso gratuito."
date: "2026-09-01"
kategorie: "Acceso remoto y soporte"
timeToRead: "9 min de lectura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-configurar-la-alternativa-de-codigo-abierto-a-teamviewer"
translationId: "article-425ae4b8d562ae41"
translationOf: rustdesk-teamviewer-alternative
translationSourceHash: 3c44a4d5fc03c777c33f77525bcaebe75596ddde249f697041599b54e77cae2b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:59:44.171Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/rustdesk-configurar-la-alternativa-de-codigo-abierto-a-teamviewer
---

# RustDesk: configurar la alternativa de código abierto a TeamViewer

TeamViewer y AnyDesk cubren el acceso remoto de forma fiable, pero requieren una licencia para uso comercial y los precios aumentan con el número de dispositivos gestionados. RustDesk es una alternativa bajo la licencia AGPL-3.0: de código abierto, gratuita y sin obligación de licencia. El cliente funciona en Windows, macOS, Linux, Android e iOS, así como en el navegador. Está escrito en Rust y su interfaz en Flutter.

La diferencia decisiva respecto a los productos comerciales está en la intermediación: RustDesk separa el cliente de la infraestructura de servidores. Puede utilizar el servidor de intermediación público gratuito, operar su propio servidor o establecer una conexión directa sin servidor de intermediación. De este modo, RustDesk puede utilizarse desde un único puesto hasta una plataforma de soporte autoalojada, sin que los datos de conexión tengan que pasar por un proveedor.

## Los tres tipos de conexión

Antes de instalarlo, debe definir el tipo de conexión, ya que de él dependen la configuración y los puertos abiertos.

| Tipo de conexión | Cómo funciona | Cuándo conviene |
|---|---|---|
| Servidor de intermediación público | Dos clientes se encuentran mediante la ID (número de nueve dígitos) en el servidor RustDesk; la conexión se realiza directamente o a través de un relay | Inicio rápido, pruebas, soporte ocasional privado |
| Servidor propio (self-hosted) | Usted opera por su cuenta los componentes de servidor `hbbs` (intermediación) y `hbbr` (relay); todos los clientes introducen su dirección | Uso comercial, muchos dispositivos, control total sobre los datos |
| Conexión directa (Direct IP Access) | El cliente se conecta directamente a la dirección IP del otro extremo sin servidor de intermediación | Ambos dispositivos pueden alcanzarse en la misma red o mediante una VPN |

El servidor público está expresamente destinado a pruebas y uso privado. Para un funcionamiento comercial y productivo, el proyecto recomienda un servidor propio, también porque el servicio público está limitado y no ofrece garantía de disponibilidad.

## Instalación en Windows

Puede descargar el instalador desde la fuente oficial, las versiones publicadas del proyecto en GitHub (`github.com/rustdesk/rustdesk`). Para Windows hay un archivo ejecutable y un paquete MSI. Para la instalación interactiva basta con hacer doble clic. Si desea distribuir RustDesk en varios equipos o en segundo plano, utilice el MSI con una instalación silenciosa. El nombre del archivo contiene el número de versión; la versión actual desde el 30 de septiembre de 2026 es la 1.5.0:

```powershell
msiexec /i rustdesk-1.5.0-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/i <paket>` | Instala el paquete MSI indicado |
| `/qn` | Sin interfaz ni diálogos (silencioso) |
| `/norestart` | Impide un reinicio automático tras la instalación |

</details>

La instalación silenciosa configura el servicio `RustDesk` para que se ejecute al iniciar el sistema y permita el acceso desatendido. Tras la instalación, puede consultar la ID del dispositivo desde la línea de comandos sin abrir la interfaz:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

También puede establecer una contraseña fija para el acceso desatendido desde la línea de comandos. Asigne una contraseña independiente y suficientemente larga; no use la contraseña de inicio de sesión del usuario:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--get-id` | Muestra la ID de RustDesk de nueve dígitos del dispositivo |
| `--password <wert>` | Establece la contraseña fija para el acceso desatendido |
| `--silent-install` | Instala la versión ejecutable (`.exe`) como servicio sin interfaz |

</details>

## Introducir un servidor propio

Si opera su propio servidor de intermediación, introduzca en los clientes su dirección y la clave pública. En la interfaz encontrará estas opciones en la configuración de red como servidor de ID, servidor relay y clave. Para la distribución masiva, también puede predefinir la configuración como archivo o mediante variables de entorno, de modo que cada cliente se inicie preconfigurado.

Un servidor propio necesita los dos componentes `hbbs` y `hbbr`, que normalmente se ejecutan como contenedores Docker. Ambos requieren puertos abiertos para que los clientes puedan registrarse y utilizar un relay.

| Puerto | Protocolo | Componente y finalidad |
|---|---|---|
| 21114 | TCP | Interfaz web de la versión Pro (solo en ella) |
| 21115 | TCP | `hbbs`, prueba del tipo de NAT |
| 21116 | TCP y UDP | `hbbs`, registro (UDP) y establecimiento de conexión (TCP) |
| 21117 | TCP | `hbbr`, tráfico de relay |
| 21118, 21119 | TCP | Compatibilidad con clientes web |

Abra únicamente los puertos que realmente requiera su tipo de conexión y restrinja mediante el firewall el acceso a las redes desde las que se presta soporte.

## Conexión directa sin servidor de intermediación

Si ambos dispositivos pueden alcanzarse en la misma red o mediante una VPN, RustDesk funciona sin servidor de intermediación. Para ello, active el acceso directo en el dispositivo de destino (en la interfaz, dentro de seguridad, como "Activar acceso directo por IP"; internamente, el conmutador `direct-server`). El cliente escucha entonces en el puerto estándar 21118 (TCP). En la ventana de conexión, introduzca la dirección IP del otro extremo en lugar de la ID.

Limite el acceso directo mediante el firewall a la red desde la que accede. Si el acceso se realiza mediante una VPN, habilite el puerto únicamente para el rango de direcciones de la VPN, no para todo Internet.

## Funciones en el día a día del soporte

RustDesk cubre las funciones que se necesitan para el acceso remoto diario:

- Transmisión de pantalla y control remoto de teclado y ratón, con selección del monitor en caso de varias pantallas.
- Transferencia de archivos en ambas direcciones mediante una ventana dividida.
- Chat de texto durante la sesión.
- Acceso desatendido mediante una contraseña fija, para dispositivos sin usuario presente.
- Grabación de sesiones como archivo de vídeo, automáticamente si se desea.
- Túneles TCP y redirección para acceder localmente a servicios individuales del otro extremo.
- Libreta de direcciones y varios dispositivos guardados, localmente en la versión gratuita y compartidos en el servidor en la versión Pro.

Para el soporte asistido, es importante lo siguiente: de forma predeterminada, RustDesk pregunta en el otro extremo si se acepta la conexión y muestra durante la sesión que hay un acceso en curso. Por tanto, la persona que utiliza el dispositivo está informada. Solo una contraseña fija para el acceso desatendido elimina la confirmación. Utilice el acceso desatendido únicamente en dispositivos cuyos usuarios sepan que el software está instalado y para qué sirve.

## Restricciones y límites

RustDesk sustituye a TeamViewer en muchos casos, pero tiene límites que debe conocer antes de utilizarlo:

- El servidor de intermediación público está limitado, no ofrece garantía de disponibilidad y no está previsto para funcionamiento comercial continuo. Quien quiera trabajar de forma fiable debe alojarlo por su cuenta.
- Un servidor propio implica esfuerzo operativo: los contenedores, los puertos abiertos, los certificados y las actualizaciones son su responsabilidad.
- Una libreta de direcciones compartida en el servidor, la gestión centralizada de usuarios y la interfaz web de administración pertenecen a la versión Pro, que es de pago a partir de cierto número de dispositivos. El cliente en sí y el funcionamiento básico siguen siendo gratuitos.
- Sin contraseña fija no es posible el acceso desatendido, lo cual es correcto para el soporte asistido, pero impide el acceso espontáneo a un dispositivo sin ocupante.
- La amplitud funcional y la estabilidad de algunas plataformas, especialmente en dispositivos móviles, no alcanzan en todos los detalles a los productos comerciales. Compruebe las funciones importantes para usted antes de cambiar.
- Algunos programas de seguridad detectan el software de acceso remoto como potencialmente no deseado. Si es necesario, configure una excepción y documente por qué está instalado el software.

Para uso privado y soporte de dispositivos individuales, basta con la versión gratuita y el servidor público o una conexión directa. En cuanto gestione muchos dispositivos, trabaje comercialmente o necesite un control total sobre los datos, necesitará un servidor propio, con el correspondiente esfuerzo operativo a cambio de independencia.

## Historial de cambios

- **25 de octubre de 2026:** Ejemplo de instalación actualizado a RustDesk 1.5.0 del 30 de septiembre de 2026.

## Fuentes

1.  [RustDesk en GitHub](https://github.com/rustdesk/rustdesk): código fuente, versiones con los instaladores y la licencia AGPL-3.0.

2.  [Documentación de RustDesk](https://rustdesk.com/docs/): instalación, servidor propio, puertos y configuración de los clientes.

3.  [rustdesk-server en GitHub](https://github.com/rustdesk/rustdesk-server): componentes de servidor `hbbs` y `hbbr`, incluida la vista general de puertos para la operación propia.
