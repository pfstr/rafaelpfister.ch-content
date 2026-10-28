---
title: "Requisitos para que funcione PowerShell remoto"
navTitle: "PowerShell remoto"
description: "La administración remota de PowerShell rara vez falla por el comando, sino por los requisitos: servicio WinRM, listener, firewall, autenticación y las particularidades de las cuentas locales. Qué debe configurarse en el destino y en el cliente, cómo comprobarlo con Test-WSMan y por qué Access denied normalmente no tiene que ver con la contraseña."
date: "2026-09-01"
kategorie: "Windows y PowerShell"
timeToRead: "10 min de lectura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "requisitos-para-que-funcione-powershell-remoto"
translationId: "article-7315c1ae9e67a24d"
translationOf: remote-powershell-voraussetzungen
translationSourceHash: edebb6e6be909c38c0d70f1bf09aaac37328f86bf2def4e23a26a34456b3492f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:57:40.690Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/requisitos-para-que-funcione-powershell-remoto
---

# Requisitos para que funcione PowerShell remoto

`Invoke-Command` y `Enter-PSSession` se escriben rápidamente, pero la conexión solo se establece cuando se cumplen los requisitos en ambos lados. La administración remota de PowerShell se basa en WS-Management (WinRM), un servicio de administración basado en SOAP a través de HTTP. Si una sesión falla, casi nunca se debe al propio cmdlet, sino a un servicio ausente, un puerto cerrado, una regla de firewall o la autenticación. Este artículo repasa los requisitos en orden y muestra cómo comprobar cada uno de ellos.

Primero, los términos: el equipo de destino es el equipo en el que deben ejecutarse los comandos; el cliente es el equipo desde el que se conecta. WinRM escucha de forma predeterminada en el puerto 5985 (HTTP) y, si está configurado, en el puerto 5986 (HTTPS). El tráfico HTTP en el puerto 5985 se cifra a nivel de mensaje en cuanto la autenticación se realiza mediante Kerberos o NTLM.

## Los cmdlets de un vistazo

Como orientación, estos son los cmdlets que aparecen en este artículo:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Cmdlet | Finalidad |
|---|---|
| `Enable-PSRemoting` | Configura WinRM en el destino: servicio, listener y regla de firewall |
| `Test-WSMan` | Comprueba si responde el servicio WinRM del equipo remoto |
| `Enter-PSSession` | Abre una sesión remota interactiva con un equipo |
| `Invoke-Command` | Ejecuta un bloque de comandos en uno o varios equipos |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Registra equipos de confianza para la autenticación fuera de un dominio |
| `Get-Service WinRM` | Muestra el estado y el tipo de inicio del servicio WinRM |

</details>

## Destino: configurar WinRM

En el equipo de destino, un único comando configura el servicio, el listener y la regla de firewall. Ejecútelo en una PowerShell con privilegios de administrador:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Force` | Se ejecuta sin pedir confirmación |
| `-SkipNetworkProfileCheck` | Configura la administración remota incluso si una conexión de red está clasificada como pública |

</details>

`Enable-PSRemoting` inicia el servicio WinRM, establece su tipo de inicio en automático, crea un listener HTTP y añade la regla de firewall correspondiente. Hay una salvedad relacionada con el perfil de red: si una tarjeta de red está clasificada como pública, el comando rechaza la configuración de forma predeterminada. En servidores o redes controladas, `-SkipNetworkProfileCheck` permite completar la configuración de todos modos.

El ámbito de la regla de firewall es importante. Para los perfiles de red públicos, la regla predeterminada limita el acceso a la subred local. Si se conecta desde otra red, por ejemplo mediante una VPN, se aplica esta limitación y la conexión falla aunque el servicio esté en ejecución. Abra entonces la regla específicamente para el rango de direcciones necesario, no de forma general para todas las direcciones:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Selecciona mediante el patrón de nombre las reglas HTTP de WinRM creadas por Enable-PSRemoting |
| `-RemoteAddress <Bereich>` | Limita las direcciones de origen permitidas al rango indicado (aquí, un bloque CIDR); `Any` permite cualquier dirección |

</details>

## Cliente: TrustedHosts y servicio

En el cliente debe estar en ejecución el servicio WinRM; de lo contrario, incluso la configuración de ajustes fallará. Compruébelo primero:

```powershell
Get-Service WinRM
```

Si el servicio aparece como Stopped, inícielo con `Start-Service WinRM` (se requieren privilegios de administrador). En los clientes, el tipo de inicio suele ser manual, por lo que el servicio vuelve a estar detenido tras reiniciar. Si accede regularmente desde este equipo, establezca el tipo de inicio en automático.

El segundo punto se refiere a la autenticación fuera de un dominio. Si se conecta mediante una dirección IP o en un grupo de trabajo, el cliente no puede comprobar el equipo remoto mediante Kerberos y recurre a NTLM. Por motivos de seguridad, WinRM lo rechaza mientras el equipo remoto no esté registrado como de confianza. Añada la dirección de destino a TrustedHosts (se requieren privilegios de administrador):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Value <Liste>` | Equipos de confianza (IP o nombre), varios separados por comas, `*` como comodín |
| `-Force` | Establece el valor sin pedir confirmación |
| `-Concatenate` | Añade a la lista existente en lugar de sustituirla |

</details>

TrustedHosts es una configuración del cliente, no del equipo de destino, y afecta a la seguridad del cliente: los equipos registrados se consideran de confianza sin que su identidad se compruebe criptográficamente. Por ello, introduzca direcciones concretas y no el comodín `*`. En un dominio con Kerberos, no es necesario registrarlo; la forma adecuada fuera de un dominio y sin TrustedHosts es usar un listener HTTPS con un certificado en el que el cliente confíe.

## Autenticación: por qué Access denied rara vez se debe a la contraseña

Un error frecuente con cuentas locales es el mensaje Access denied, aunque la contraseña sea correcta. La razón es el filtrado UAC remoto: en el caso de las cuentas locales (excepto el administrador integrado), Windows elimina de forma predeterminada los privilegios administrativos para el acceso a través de la red. El inicio de sesión funciona, pero se rechaza cualquier acción que requiera privilegios elevados. Si el equipo remoto informa Access denied en lugar de credenciales incorrectas, esta es la causa probable.

Puede corregirse en el equipo de destino con un valor del Registro que otorga a los administradores locales todos los privilegios a través de la red:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Esta es una relajación deliberada: las cuentas de administrador local obtienen así todos los privilegios a través de la red. Establezca este valor solo en redes controladas y con contraseñas seguras. En un dominio, es preferible utilizar una cuenta de dominio, con lo que este problema no se plantea.

Al establecer la conexión, introduzca el nombre de usuario de las cuentas locales precedido por el nombre del equipo, para que el sistema de destino resuelva la cuenta localmente:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

En el cuadro de inicio de sesión, introduzca el usuario como `RECHNERNAME\Benutzer` y, para las cuentas de dominio, como `DOMAENE\Benutzer`. Un PIN de inicio de sesión de Windows no funciona a través de la red; se necesita la contraseña de la cuenta. En el caso de una cuenta Microsoft, se trata de su contraseña y el nombre de la cuenta puede diferir del nombre para mostrar.

## Comprobar en el orden correcto

Aísle los errores de abajo arriba; así verá rápidamente qué requisito falta.

Primero, compruebe la accesibilidad del puerto:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Si el puerto no responde, falta el listener o el firewall bloquea el acceso. Si responde, compruebe el servicio WinRM del equipo remoto:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Una respuesta con la versión del protocolo y el fabricante significa que el servicio y el listener están activos. Solo entonces pruebe con credenciales:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Si esta llamada devuelve el nombre del equipo remoto, se cumplen todos los requisitos.

## Errores habituales y sus causas

| Mensaje o síntoma | Causa probable | Enfoque |
|---|---|---|
| Puerto 5985 inaccesible | No hay listener o el firewall bloquea el acceso | Comprobar `Enable-PSRemoting`, la regla de firewall y su ámbito |
| WinRM cannot complete the operation | El servicio del destino está detenido o el acceso solo se permite desde la subred local | Iniciar el servicio y abrir la regla de firewall para el rango de direcciones necesario |
| The WinRM client cannot process the request … TrustedHosts | Conexión fuera de dominio sin entrada en TrustedHosts | Añadir la dirección de destino a TrustedHosts en el cliente o usar HTTPS |
| Access is denied (a pesar de la contraseña correcta) | Filtrado UAC remoto con una cuenta local | Establecer `LocalAccountTokenFilterPolicy` en 1 o utilizar una cuenta de dominio |
| El acceso a un segundo recurso falla en la sesión | Double hop: las credenciales no se reenvían | Ejecutar la tarea directamente en el destino o utilizar CredSSP o una autenticación externa |

## Límites: el problema del double hop

Incluso con una configuración completa, permanece una limitación que solo puede sortearse: de forma predeterminada, una sesión remota no puede reenviar sus credenciales a un tercer sistema. Si accede desde una sesión en el equipo de destino a un recurso compartido de red u otro servidor, el acceso falla por falta de credenciales. Este problema de double hop es una característica de seguridad, no una configuración errónea. Para la mayoría de las tareas de soporte, basta con ejecutar el comando directamente en el equipo de destino. Cuando el reenvío es realmente necesario, entran en juego CredSSP o una delegación restringida, ambos con sus propias consideraciones de seguridad.

## Historial de cambios

- **25 de octubre de 2026:** Se actualizó el enlace de Fuentes a la nueva dirección de Microsoft Learn.

## Fuentes

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): requisitos para la administración remota de PowerShell, permisos y perfiles de red.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): qué configura el comando, incluida la salvedad del perfil de red y la regla de firewall.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autenticación fuera del dominio y mensajes de error habituales.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop): causa del problema de double hop y las soluciones con sus consideraciones.
