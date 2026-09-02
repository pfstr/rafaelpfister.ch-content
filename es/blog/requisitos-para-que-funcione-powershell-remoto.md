---
title: "Requisitos para que funcione PowerShell remoto"
navTitle: "PowerShell remoto"
description: "La comunicación remota de PowerShell rara vez falla por el comando, sino por los requisitos previos: servicio WinRM, listener, firewall, autenticación y las particularidades de las cuentas locales. Qué debe configurarse en el destino y en el cliente, cómo comprobarlo con Test-WSMan y por qué Access denied normalmente no tiene que ver con la contraseña."
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
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/es/blog/requisitos-para-que-funcione-powershell-remoto
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:46:51.092Z
translationReview: automatic
---

# Requisitos para que funcione PowerShell remoto

`Invoke-Command` y `Enter-PSSession` se escriben rápidamente, pero la conexión solo se establece cuando se cumplen los requisitos en ambos lados. La comunicación remota de PowerShell se basa en WS-Management (WinRM), un servicio de administración basado en SOAP a través de HTTP. Si falla una sesión, casi nunca se debe al propio cmdlet, sino a un servicio que falta, un puerto cerrado, una regla de firewall o la autenticación. Este artículo repasa los requisitos en orden y muestra cómo comprobar cada uno de ellos.

Primero, los términos: el equipo de destino es aquel en el que deben ejecutarse los comandos; el cliente es el equipo desde el que se conecta. WinRM escucha de forma predeterminada en el puerto 5985 (HTTP) y, si está configurado, en el puerto 5986 (HTTPS). El tráfico HTTP en el puerto 5985 se cifra en el nivel de mensaje en cuanto la autenticación se realiza mediante Kerberos o NTLM.

## Resumen de los cmdlets

Como orientación, estos son los cmdlets que aparecen en este artículo:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Cmdlet | Finalidad |
|---|---|
| `Enable-PSRemoting` | Configura WinRM en el destino: servicio, listener y regla de firewall |
| `Test-WSMan` | Comprueba si responde el servicio WinRM del equipo remoto |
| `Enter-PSSession` | Abre una sesión remota interactiva con un equipo |
| `Invoke-Command` | Ejecuta un bloque de comandos en uno o varios equipos |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Añade equipos remotos de confianza para la autenticación fuera de un dominio |
| `Get-Service WinRM` | Muestra el estado y el tipo de inicio del servicio WinRM |

</details>

## Destino: configurar WinRM

En el equipo de destino, un único comando configura el servicio, el listener y la regla de firewall. Ejecútelo en una PowerShell con derechos de administrador:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Force` | Ejecuta sin solicitar confirmación |
| `-SkipNetworkProfileCheck` | Configura la comunicación remota incluso si una conexión de red está clasificada como pública |

</details>

`Enable-PSRemoting` inicia el servicio WinRM, establece su tipo de inicio en automático, crea un listener HTTP y añade la regla de firewall correspondiente. Hay una salvedad respecto al perfil de red: si una tarjeta de red está clasificada como pública, el comando rechaza la configuración de forma predeterminada. En servidores o redes controladas, `-SkipNetworkProfileCheck` permite que la configuración se complete de todos modos.

Es importante el ámbito de la regla de firewall. Para los perfiles de red públicos, la regla estándar limita el acceso a la subred local. Si se conecta desde otra red, por ejemplo mediante una VPN, se aplica esta limitación y la conexión falla aunque el servicio esté en ejecución. En ese caso, abra la regla específicamente para el rango de direcciones necesario, no de forma general para todas las direcciones:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Selecciona las reglas WinRM HTTP creadas por Enable-PSRemoting mediante el patrón de nombre |
| `-RemoteAddress <Bereich>` | Limita las direcciones de origen permitidas al rango indicado (aquí, un bloque CIDR); `Any` permite cualquier dirección |

</details>

## Cliente: TrustedHosts y servicio

En el cliente debe estar ejecutándose el servicio WinRM; de lo contrario, incluso la configuración de ajustes fallará. Compruébelo primero:

```powershell
Get-Service WinRM
```

Si el servicio aparece como Stopped, inícielo con `Start-Service WinRM` (se requieren derechos de administrador). En los clientes, el tipo de inicio suele ser manual, por lo que el servicio vuelve a quedar detenido después de reiniciar. Si accede regularmente desde este equipo, establezca el tipo de inicio en automático.

El segundo punto se refiere a la autenticación fuera de un dominio. Si se conecta mediante una dirección IP o en un grupo de trabajo, el cliente no puede verificar el equipo remoto mediante Kerberos y recurre a NTLM. Por razones de seguridad, WinRM lo rechaza mientras el equipo remoto no esté registrado como de confianza. Añada la dirección de destino a TrustedHosts (se requieren derechos de administrador):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Value <Liste>` | Equipos remotos de confianza (IP o nombre), varios separados por comas, `*` como comodín |
| `-Force` | Establece el valor sin solicitar confirmación |
| `-Concatenate` | Añade a la lista existente en lugar de reemplazarla |

</details>

TrustedHosts es una configuración del cliente, no del equipo de destino, y afecta a la seguridad del cliente: los equipos remotos registrados se consideran de confianza sin que su identidad se compruebe criptográficamente. Por ello, introduzca direcciones concretas y no el comodín `*`. En un dominio con Kerberos, la entrada no es necesaria; la forma adecuada fuera de un dominio sin TrustedHosts es un listener HTTPS con un certificado en el que confíe el cliente.

## Autenticación: por qué Access denied rara vez se debe a la contraseña

Un error frecuente con cuentas locales es el mensaje Access denied, aunque la contraseña sea correcta. El motivo es el filtrado remoto de UAC: en las cuentas locales (excepto el administrador integrado), Windows elimina de forma predeterminada los derechos administrativos al acceder a través de la red. El inicio de sesión se realiza correctamente, pero se rechaza cualquier acción con privilegios elevados. Si el equipo remoto informa Access denied en lugar de credenciales incorrectas, esta es la causa probable.

Puede solucionarse en el equipo de destino con un valor del Registro que concede a los administradores locales todos los derechos a través de la red:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Se trata de una relajación deliberada: las cuentas de administrador local obtienen así todos los derechos a través de la red. Establezca este valor solo en redes controladas y con contraseñas seguras. En un dominio, es preferible utilizar una cuenta de dominio; entonces esta cuestión no se plantea.

Al establecer la conexión, indique en las cuentas locales el nombre de usuario precedido por el nombre del equipo, para que el sistema de destino resuelva la cuenta localmente:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

En el cuadro de inicio de sesión, introduzca el usuario como `RECHNERNAME\Benutzer` y, para las cuentas de dominio, como `DOMAENE\Benutzer`. Un PIN del inicio de sesión de Windows no funciona a través de la red; se necesita la contraseña de la cuenta. En una cuenta Microsoft, es su contraseña, y el nombre de la cuenta puede diferir del nombre para mostrar.

## Comprobar en el orden correcto

Aísle los errores de abajo arriba; así verá rápidamente qué requisito falta.

Primero, la accesibilidad del puerto:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Si el puerto no responde, falta el listener o el firewall bloquea la conexión. Si responde, compruebe el servicio WinRM del equipo remoto:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Una respuesta con la versión del protocolo y el fabricante significa que el servicio y el listener están activos. Solo entonces pruebe con credenciales:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Si esta llamada devuelve el nombre del equipo remoto, se cumplen todos los requisitos.

## Errores habituales y su causa

| Mensaje o síntoma | Causa probable | Enfoque |
|---|---|---|
| Puerto 5985 no accesible | No hay listener o el firewall bloquea | Comprobar `Enable-PSRemoting`, la regla de firewall y su ámbito |
| WinRM cannot complete the operation | El servicio del destino está detenido o el acceso solo está permitido desde la subred local | Iniciar el servicio, abrir la regla de firewall para el rango de direcciones necesario |
| The WinRM client cannot process the request … TrustedHosts | Conexión fuera de dominio sin entrada en TrustedHosts | Añadir la dirección de destino a TrustedHosts en el cliente o utilizar HTTPS |
| Access is denied (a pesar de la contraseña correcta) | Filtrado remoto de UAC con una cuenta local | Establecer `LocalAccountTokenFilterPolicy` en 1 o utilizar una cuenta de dominio |
| El acceso a un segundo recurso falla en la sesión | Double hop: las credenciales no se reenvían | Ejecutar la tarea directamente en el destino o utilizar CredSSP o credenciales delegadas |

## Límites: el problema del double hop

Incluso con una configuración completa, queda una limitación que solo puede evitarse: de forma predeterminada, una sesión remota no puede reenviar sus credenciales a un tercer sistema. Si, en una sesión en el equipo de destino, accede a un recurso compartido de red o a otro servidor, la operación falla por falta de credenciales. Este problema de double hop es una característica de seguridad, no una configuración incorrecta. Para la mayoría de las tareas de soporte, basta con ejecutar el comando directamente en el equipo de destino. Cuando el reenvío es realmente necesario, entran en juego CredSSP o la delegación restringida, ambos con sus propias consideraciones de seguridad.

## Fuentes

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): requisitos para la comunicación remota de PowerShell, derechos y perfiles de red.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): qué configura el comando, incluida la salvedad del perfil de red y la regla de firewall.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autenticación fuera del dominio y los mensajes de error habituales.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): causa del problema de double hop y los enfoques de solución con sus consideraciones.
