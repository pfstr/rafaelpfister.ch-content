---
title: "PowerShell: canalización de objetos, tiempo de ejecución y administración segura"
blatt: "powershell"
description: "PowerShell para administradores de infraestructura y mensajería: ediciones y tiempo de ejecución de .NET, host, motor, runspace y canalización de objetos, módulos y proveedores, acceso remoto mediante WSMan y SSH, autenticación y JEA, semántica de errores y cambios, paralelización, registro, historia técnica y automatización lista para producción."
fakten:
  - label: Categoría
    wert: Shell · lenguaje de scripting · marco de automatización y configuración
    href: https://learn.microsoft.com/en-us/powershell/scripting/overview
  - label: Pila tecnológica
    wert: C# · .NET · System.Management.Automation · programas nativos y API del sistema operativo
    href: https://github.com/PowerShell/PowerShell
  - label: Ediciones
    wert: Desktop sobre .NET Framework · Core sobre .NET moderno
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions
  - label: Proceso
    wert: powershell.exe para Windows PowerShell · pwsh para PowerShell 6 y posteriores
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh
  - label: Modelo de datos
    wert: Objetos .NET con tipo, propiedades y métodos en lugar de exclusivamente texto
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects
  - label: Ejecución
    wert: Host → motor → runspace → canalización → procesador de comandos
    href: https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples
  - label: Extensión
    wert: Módulos de script, binarios y de manifiesto mediante PSModulePath
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules
  - label: Acceso remoto
    wert: WSMan/WinRM o SSH; el código se ejecuta en un runspace remoto
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements
  - label: Objetos remotos
    wert: normalmente instantáneas serializadas; los métodos se pierden durante la deserialización
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output
  - label: Execution Policy
    wert: Función de seguridad de Windows contra la ejecución involuntaria, no una frontera de seguridad
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies
  - label: Delegación
    wert: JEA restringe la identidad, el endpoint y los comandos permitidos
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview
  - label: Evidencia operativa
    wert: Versión de código · resolución de módulos · identidad · entrada · cambio · resultado · registros
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/security-features
werbung:
  - newsletter
ctaThemen:
  - powershell
  - microsoft-365-exchange
  - automatisierung
translationSourceHash: a8375bc58323f55f2e87dfe08969d32b9ae12af81cbc67673940011c3245a20a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:39:17.315Z
translationReview: automatic
---

# PowerShell: canalización de objetos, tiempo de ejecución y administración segura

PowerShell es a la vez una shell interactiva, un lenguaje de scripting y un marco de integración. Esto la distingue tanto de una línea de comandos pura como de un lenguaje de programación convencional: los comandos entregan objetos tipados a una canalización, las aplicaciones pueden integrar el motor en sus propios procesos y los productos de administración publican su interfaz de gestión como cmdlets, proveedores o API HTTP. Microsoft describe PowerShell en consecuencia como un marco multiplataforma de automatización y configuración para datos estructurados, interfaces REST y modelos de objetos ([Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)).

Para la operación, por tanto, no solo es relevante la sintaxis. Son decisivos la edición concreta y el tiempo de ejecución de .NET, el contexto de proceso y runspace, los módulos cargados, la identidad en el sistema de destino, el transporte y la autorización, la semántica de errores, así como la evidencia de lo que un script ha modificado realmente. Un comando que funciona en una sesión interactiva todavía no constituye una automatización fiable.

La explicación sigue un comando a través del analizador, el motor, el runspace y la canalización de objetos hasta el sistema al que se dirige. Después, los módulos, proveedores, acceso remoto, paralelismo, seguridad y operación se relacionan con este modelo de ejecución.

PowerShell es simultáneamente shell, lenguaje de automatización y tiempo de ejecución de administración. Lo decisivo es que los cmdlets transmiten objetos .NET por una canalización y que los módulos de producto generan a partir de ellos llamadas API locales o remotas. El artículo sigue este recorrido desde el comando hasta su efecto en el sistema de destino.

## Arquitectura del tiempo de ejecución

Una ejecución de PowerShell consta de varias capas:

| Capa | Tarea | Estado relevante para la administración |
|---|---|---|
| Host | proporciona consola, editor, servicio o aplicación integrada | nombre del host, interactivo/no supervisado, entrada/salida, perfil |
| Motor | analiza el lenguaje, enlaza parámetros y coordina comandos | edición, versión, .NET, modo de lenguaje, configuración |
| Runspace | contexto de ejecución aislado con estado de sesión | identidad, variables, funciones, módulos, unidades, endpoint |
| Canalización | conecta el procesador de comandos y transporta objetos/streams | orden, enlace, enumeración, flujos de error e información |
| Comando | cmdlet, función, script, alias o programa nativo | origen, versión del módulo, parámetros, efectos secundarios |
| Adaptador/proveedor/API | traduce a .NET, sistema operativo, almacén o servicio remoto | endpoint, autenticación, permisos, límites de tasa, ámbito de fallo |

El **host** no es el motor. Windows Terminal, la consola clásica, Visual Studio Code, una tarea programada o una aplicación integrada en C# pueden operar el mismo motor de PowerShell de formas distintas. Un **runspace** es el entorno operativo en el que se ejecutan los comandos de una canalización; varios runspaces dentro de un proceso pueden proporcionar distintos estados de sesión y paralelismo ([PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary), [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-powershell.svg?v=20260813" title="Interaktive Infografik: PowerShell-Laufzeit von Host, Engine und Runspace über Objektpipeline, Module, Provider und APIs bis zu Remoting, Vertrauensgrenzen und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-powershell.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

El nombre del proceso es una característica de diagnóstico importante: `powershell.exe` inicia Windows PowerShell, y `pwsh` la línea de PowerShell basada en .NET moderno. Ambos pueden instalarse en paralelo en Windows y, por ello, ver módulos, perfiles, valores predeterminados de TLS y ensamblados diferentes. La interfaz de línea de comandos de `pwsh` también documenta cómo se enlazan archivos, comandos, codificaciones y llamadas no interactivas ([Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)).

### Inventariar el tiempo de ejecución y el contexto

[$PSVersionTable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) muestra la edición, el motor y la plataforma; [`Get-Host`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host) describe el host, no necesariamente la versión subyacente del sistema operativo. Para un informe de error reproducible deben incluirse al menos la ruta del proceso, edición, versión, descripción de .NET, arquitectura, modo de lenguaje y ruta de módulos.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Laufzeitinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process -Id $PID | Select-Object Path,Id,StartTime
$PSVersionTable
Get-Host
[System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
$ExecutionContext.SessionState.LanguageMode
$env:PSModulePath -split [IO.Path]::PathSeparator</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Process -Id $PID | Select-Object Path,Id,StartTime
  $PSVersionTable
  Get-Host
  [System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
  $ExecutionContext.SessionState.LanguageMode
  $env:PSModulePath -split [IO.Path]::PathSeparator
'</code></pre>
  </div>
</div>

## Ediciones, .NET y compatibilidad

Desde PowerShell 5.1, Microsoft distingue las ediciones **Desktop** y **Core**. Desktop se ejecuta sobre .NET Framework específico de Windows; Core se ejecuta sobre .NET moderno y está disponible para Windows, Linux y macOS. Sin embargo, el campo `CompatiblePSEditions` de un manifiesto de módulo ajeno es solo metadato: PowerShell no aplica automáticamente la compatibilidad allí declarada ([Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)).

Por ello, Windows PowerShell 5.1 y PowerShell 7+ son dos productos independientes, no dos opciones de inicio de la misma instalación. PowerShell 7 no puede reemplazar Windows PowerShell in situ; los perfiles, rutas de módulos y partes de la superficie API son diferentes. La guía de migración de Microsoft recomienda probar explícitamente las dependencias y la compatibilidad de los módulos ([Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)).

La decisión de compatibilidad se toma por carga de trabajo:

- **Motor:** ¿el script requiere Windows PowerShell/Desktop o PowerShell/Core?
- **Sistema operativo:** ¿invoca cmdlets exclusivos de Windows, COM, Registry, WMI o programas nativos?
- **Módulo:** ¿es portátil como módulo de script, está construido como módulo binario para el tiempo de ejecución objetivo de .NET correcto o solo puede utilizarse mediante un proceso de compatibilidad?
- **Autenticación:** ¿el tiempo de ejecución admite el flujo requerido, el almacén de certificados y la caché de tokens?
- **Serialización:** ¿el objeto necesario sigue vivo localmente o llega de otro proceso como una instantánea deserializada?
- **Soporte:** ¿el fabricante indica exactamente esta combinación de motor, módulo y producto?

PowerShell 7 puede cargar determinados módulos de Windows PowerShell mediante una sesión implícita de acceso remoto/compatibilidad. Sin embargo, esto genera objetos deserializados y un límite de proceso adicional; no sustituye una prueba de compatibilidad nativa ([Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)).

Una vez definidas la edición y el tiempo de ejecución, la canalización de objetos explica la diferencia más importante frente a las shells orientadas a texto: el filtrado, la ordenación y la exportación trabajan con propiedades en lugar de posiciones de columnas.

## La canalización de objetos

Una canalización Unix clásica conecta flujos de bytes o texto. PowerShell transmite fundamentalmente objetos .NET dentro de un proceso. Un objeto posee tipo, propiedades y métodos; el siguiente componente de la canalización enlaza entradas según reglas de parámetros y puede acceder a propiedades sin volver a analizar la salida tabular ([Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects), [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)).

La tabla visible no es el objeto. El sistema de formato selecciona vistas y genera instrucciones de formato solo al final de la canalización. `Format-Table` y `Format-List` deben situarse, por tanto, en el extremo de salida; si se usan en medio de una canalización de datos, los comandos posteriores ya no reciben los objetos de administración originales ([Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Objektpipeline">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$service = Get-Service -Name WinRM
$service.GetType().FullName
$service | Get-Member
$service | Select-Object Name,Status,StartType
$service | ConvertTo-Json -Depth 3</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  $process = Get-Process -Id $PID
  $process.GetType().FullName
  $process | Get-Member
  $process | Select-Object Id,ProcessName,Path
  $process | ConvertTo-Json -Depth 3
'</code></pre>
  </div>
</div>

### Enlace, enumeración y streams

El enlace de canalización se produce por parámetro, por valor o por nombre de propiedad. Las colecciones se enumeran normalmente elemento a elemento; esto modifica el comportamiento y el consumo de memoria en comparación con una lista única. Para funciones fiables, los parámetros de canalización se declaran explícitamente, la entrada se procesa en el bloque `process` y la salida no se contamina accidentalmente con valores auxiliares.

PowerShell ejecuta varios streams lógicos: Success, Error, Warning, Verbose, Debug, Information y Progress. Solo el stream Success es la salida normal de objetos; los errores pueden ser visibles sin detener la canalización. La redirección y combinación de streams forman parte de la interfaz de llamada y deben definirse conscientemente en tareas programadas, CI y monitorización ([Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)).

Los programas nativos siguen siendo un punto de transición. Sus comillas de argumentos, codificación de caracteres, salida estándar y códigos de salida no siguen por completo el modelo de cmdlets. Un wrapper debe tratar `$LASTEXITCODE`, los códigos de salida esperados, la codificación y stderr, y normalizar el resultado como su propio objeto. La mera existencia de texto en stderr no equivale en todos los programas a un fallo.

## Resolución de comandos, módulos y proveedores

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command) muestra qué alias, función, cmdlet, script o programa nativo se invoca realmente para un nombre. Esto es más importante que la sintaxis visible: un alias o función del perfil con el mismo nombre puede ocultar un cmdlet. La llamada totalmente cualificada `Modulname\\Verb-Noun` hace explícito el origen.

Los módulos empaquetan comandos, datos de tipo y formato, así como dependencias. Un módulo de script suele usar `.psm1`, un módulo binario usa un ensamblado .NET; un manifiesto `.psd1` describe, entre otras cosas, el módulo raíz, la versión, los comandos exportados, los requisitos y las ediciones compatibles. Un manifiesto no es obligatorio para la carga, pero sí para la publicación en PowerShell Gallery ([Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules), [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)).

El motor busca módulos mediante `PSModulePath`. Es posible tener varias versiones instaladas; la carga automática y llamadas imprecisas a `Import-Module` pueden hacer que un servidor elija una implementación diferente a la del entorno de pruebas. Por ello, un script de producción inventaría el nombre, versión, ruta y repositorio del módulo, y fija las dependencias en el despliegue en vez de instalarlas sin control durante cada ejecución ([Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Modulinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command Get-Mailbox -All |
  Select-Object Name,CommandType,Source,Version,Definition
Get-Module -ListAvailable |
  Sort-Object Name,Version |
  Select-Object Name,Version,Path,CompatiblePSEditions
Get-InstalledModule -ErrorAction SilentlyContinue |
  Select-Object Name,Version,Repository</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Command Invoke-RestMethod -All |
    Select-Object Name,CommandType,Source,Version,Definition
  Get-Module -ListAvailable |
    Sort-Object Name,Version |
    Select-Object Name,Version,Path,CompatiblePSEditions
  Get-InstalledModule -ErrorAction SilentlyContinue |
    Select-Object Name,Version,Repository
'</code></pre>
  </div>
</div>

Los **proveedores** proporcionan almacenes de datos jerárquicos con una interfaz similar a un sistema de archivos. `FileSystem:`, `Registry:`, `Certificate:`, `Variable:`, `Function:` o `WSMan:` parecen unidades, pero tienen capacidades y efectos secundarios distintos. Antes de escribir código genérico, `Get-PSProvider` muestra qué operaciones admite un proveedor. La especificación histórica del lenguaje menciona proveedores y unidades como modelo fundamental para datos de difícil acceso ([Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)).

Un comando local se ejecuta en el proceso actual; el acceso remoto traslada la ejecución a un runspace del destino. Con ello cambian la identidad, la ruta de red, los módulos y el acceso a los recursos locales.

## Acceso remoto: cliente local, transporte y runspace remoto

PowerShell Remoting ejecuta el código fundamentalmente en el lado de destino. Localmente se preparan el comando y los argumentos; el transporte, el listener y la configuración de sesión autentican la llamada y crean un runspace remoto. Su sistema operativo, motor, módulos, políticas, identidad y acceso a red determinan el resultado.

En Windows, el acceso remoto clásico utiliza WS-Management mediante WinRM. PowerShell 7 también admite acceso remoto SSH en Windows, Linux y macOS. Ambos transportes llevan datos de PowerShell Remoting, pero no ofrecen las mismas funciones de endpoint: Microsoft señala, por ejemplo, que el acceso remoto SSH no dispone de configuración de endpoint de PowerShell y, por tanto, no admite JEA ([Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements), [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Remoting">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-WSMan -ComputerName mail01.example.ch
$session = New-PSSession -ComputerName mail01.example.ch -ConfigurationName Microsoft.PowerShell
Invoke-Command -Session $session -ScriptBlock {
  [pscustomobject]@{
    Host = $env:COMPUTERNAME
    Edition = $PSVersionTable.PSEdition
    Version = $PSVersionTable.PSVersion.ToString()
  }
}
Remove-PSSession $session</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh -G admin@mail01.example.ch | grep -E '^(hostname|user|port|identityfile) '
pwsh -NoProfile -Command '
  $s = New-PSSession -HostName mail01.example.ch -UserName admin
  Invoke-Command -Session $s -ScriptBlock {
    [pscustomobject]@{
      Host = [Environment]::MachineName
      Edition = $PSVersionTable.PSEdition
      Version = $PSVersionTable.PSVersion.ToString()
    }
  }
  Remove-PSSession $s
'</code></pre>
  </div>
</div>

Las herramientas utilizadas están documentadas en [`Test-WSMan`](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman), [`New-PSSession`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession), [`Invoke-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) y en la página de manual de OpenSSH para [`ssh`](https://man.openbsd.org/ssh).

### Serialización y pérdida de objetos

La mayoría de los objetos remotos se serializan en el destino y se reconstruyen localmente. El resultado es una instantánea con propiedades, pero normalmente sin los métodos del objeto activo original; el nombre de tipo suele llevar el prefijo `Deserialized.`. Los datos de formato y la profundidad de tipo determinan qué propiedades se transfieren. Los cambios en la instantánea local no modifican el objeto de destino ([Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)).

Por ello, los filtros, la agregación y los datos sensibles se procesan preferiblemente **en el destino**, y solo se transfiere el resultado necesario. Esto reduce la latencia, el volumen de serialización y la fuga de datos:

```text
schlecht:  200'000 Remoteobjekte → serialisieren → übertragen → lokal filtern
gut:       remote filtern/projizieren/aggregieren → 200 Ergebnisobjekte übertragen
```

El conocido «Second Hop» no es una particularidad de PowerShell, sino un límite de delegación: la identidad que se autentica desde el cliente de administración en el servidor A no posee automáticamente credenciales reutilizables para el servidor B. Soluciones como Kerberos Constrained Delegation, CredSSP o una cuenta de servicio modifican el modelo de confianza y solo se utilizan con un análisis explícito de amenazas ([Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)).

## Autenticación, autorización y mínimo privilegio

El acceso remoto separa cuatro cuestiones:

1. **Confianza en el transporte:** ¿con qué host se comunica el cliente y cómo se verifica su clave o certificado?
2. **Autenticación:** ¿qué persona, identidad administrada, aplicación, certificado o clave SSH inicia sesión?
3. **Autorización:** ¿qué configuración de sesión, rol y permiso de producto permite qué operaciones?
4. **Identidad de proceso:** ¿con qué cuenta local o virtual se ejecuta realmente el comando remoto?

**Just Enough Administration (JEA)** crea endpoints de PowerShell restringidos. Las capacidades de rol definen los cmdlets, funciones, proveedores y programas externos visibles; las configuraciones de sesión asignan usuarios o grupos a los roles. Las cuentas virtuales o gMSA pueden ejecutar acciones privilegiadas sin proporcionar al usuario que llama una cuenta de administrador permanente. Las transcripciones y los registros respaldan la trazabilidad ([Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)).

JEA no limita automáticamente todas las rutas de evasión. Los comandos visibles, parámetros, proveedores, acceso a archivos, tipos .NET invocables y programas externos se verifican conjuntamente. Un editor, compilador o bloque de script genérico aparentemente inocuo puede ampliar el conjunto de comandos previsto.

## Execution Policy, firmas y control de aplicaciones

La Execution Policy de PowerShell es una función de seguridad de Windows contra la ejecución accidental de scripts, **no una frontera de seguridad**. Un usuario que puede ejecutar comandos puede invocar de otra forma el contenido de un script bloqueado. En sistemas que no son Windows, la política no se aplica como en Windows ([Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)).

`AllSigned` y `RemoteSigned` utilizan Authenticode para comprobar el origen y la integridad de determinados tipos de archivo de PowerShell. Sin embargo, una firma válida solo indica que el archivo no ha cambiado desde la firma y que corresponde a una ruta de certificados de confianza; no demuestra que el código sea seguro o técnicamente correcto ([Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)).

Una cadena de control fiable combina:

- control de código fuente, revisión y artefactos de compilación inmutables;
- comprobación de firma y hash;
- repositorios de módulos controlados y versiones de dependencias;
- App Control/System Lockdown o control de aplicaciones equivalente;
- identidad de mínimo privilegio y endpoints de acceso remoto restringidos;
- secretos fuera del código fuente;
- Script Block/Module Logging, transcripciones y envío centralizado de registros.

Microsoft clasifica Execution Policy como defensa en profundidad, mientras que App Control en System Lockdown puede determinar el modo de lenguaje por runspace. Script Block Logging escribe comandos, funciones y scripts procesados en el PowerShell Operational Log; un registro más intenso puede, a su vez, capturar secretos ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

### Comprobar firma y artefacto

[`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) comprueba el estado de Authenticode en Windows; [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparan un artefacto con un valor objetivo publicado independientemente.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$script = 'C:\\Ops\\Deploy-MailPolicy.ps1'
Get-AuthenticodeSignature $script |
  Select-Object Status,StatusMessage,SignerCertificate,TimeStamperCertificate
Get-FileHash $script -Algorithm SHA256
Get-ExecutionPolicy -List</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum /opt/ops/Deploy-MailPolicy.ps1
pwsh -NoProfile -Command '
  Get-FileHash /opt/ops/Deploy-MailPolicy.ps1 -Algorithm SHA256
  Get-ExecutionPolicy -List
'</code></pre>
  </div>
</div>

## Secretos e identidades no interactivas

Un `SecureString` no es un vault. Microsoft recomienda para los desarrollos nuevos sustituir las contraseñas, siempre que sea posible, por certificados, autenticación de Windows u otros métodos sin contraseña. El módulo SecretManagement proporciona una interfaz de cmdlets uniforme para vaults de extensión; el vault registrado implementa el almacenamiento y la autenticación. El registro está vinculado al usuario, lo que debe tenerse en cuenta para cuentas de servicio y automatización sin interfaz ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features), [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)).

Para cada automatización se documentan por separado:

- **Identidad de carga de trabajo:** cuenta de servicio, gMSA, identidad administrada, registro de aplicación o certificado;
- **Origen de credenciales:** vault, almacén de certificados, federación de carga de trabajo o broker interactivo;
- **Permiso:** rol local, rol RBAC, ámbito de API o permiso de aplicación;
- **Ciclo de vida:** emisión, rotación, vencimiento, revocación y Break Glass;
- **Lugar de ejecución:** agente, tarea programada, runner, función o cliente de administración;
- **Ruta de red:** DNS, proxy, confianza TLS, endpoint y límite de tasa;
- **Ancla de auditoría:** ID de ejecución, ID de identidad, commit de código e ID de cambio/ticket.

Las variables de entorno y de canalización son mecanismos de transporte, no almacenamiento seguro. Pueden llegar a procesos secundarios, volcados, salidas de diagnóstico o registros de CI. Incluso el acceso a un vault se limita al ámbito mínimo de secretos y a una vida útil breve.

La automatización segura debe detectar si una operación ha fallado realmente. Un mensaje rojo, una excepción, un error no terminante y un código de salida nativo son señales distintas.

## Semántica de errores: una línea roja visible no es un modelo de salida

PowerShell distingue entre errores no terminantes, terminantes a nivel de instrucción y terminantes a nivel de script. Muchos cmdlets escriben un `ErrorRecord` no terminante para un objeto individual y continúan procesando el resto de la canalización. Un `try/catch` solo captura errores que abandonan el bloque controlado; `-ErrorAction Stop` puede elevar errores no terminantes adecuados para esa llamada ([Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling), [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)).

Por ello, el código de producción define:

- qué errores se toleran y registran por objeto;
- qué errores interrumpen toda la ejecución;
- qué código de salida proporciona el proceso al planificador o a CI;
- si un reintento es seguro e idempotente;
- qué ID de correlación conecta el error, el objeto de destino y la solicitud API;
- qué recursos se cierran en el bloque `finally`;
- cómo se detectan y compensan los cambios parciales.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Fehlerbehandlung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$ErrorActionPreference = 'Stop'
try {
  $result = Invoke-RestMethod -Uri $endpoint -Method Get -TimeoutSec 30
  $result | ConvertTo-Json -Depth 10
}
catch {
  Write-Error -ErrorRecord $_
  exit 1
}
finally {
  Remove-Variable result -ErrorAction SilentlyContinue
}</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -NonInteractive -Command '
  $ErrorActionPreference = "Stop"
  try {
    $result = Invoke-RestMethod -Uri $env:OPS_ENDPOINT -Method Get -TimeoutSec 30
    $result | ConvertTo-Json -Depth 10
  } catch {
    Write-Error -ErrorRecord $_
    exit 1
  } finally {
    Remove-Variable result -ErrorAction SilentlyContinue
  }
'</code></pre>
  </div>
</div>

El ejemplo utiliza [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), cuyo comportamiento HTTP y de errores debe integrarse deliberadamente en el tratamiento de errores. En programas nativos se comprueba adicionalmente `$LASTEXITCODE`. La variable de proceso `$?` y los flujos de errores de PowerShell no sustituyen un contrato documentado sobre los códigos de salida del programa invocado.

## Semántica segura de cambios e idempotencia

Una automatización de administración separa **determinar, planificar, modificar y verificar**. Compara el estado actual y el deseado mediante ID estables, genera un conjunto explícito de cambios y ejecuta solo la diferencia necesaria. Después vuelve a leer el estado mediante la interfaz determinante. «El cmdlet no devolvió ningún error» no prueba el estado.

Las funciones avanzadas pueden ofrecer los parámetros comunes `-WhatIf` y `-Confirm` con `SupportsShouldProcess`. La función invoca `$PSCmdlet.ShouldProcess()` inmediatamente antes del efecto secundario. Sin embargo, Microsoft documenta límites en la propagación de `WhatIfPreference` entre módulos de script; por ello, un `-WhatIf` correcto es una vista previa útil, no una garantía transaccional ([Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess), [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)).

Un flujo de cambio robusto conserva:

1. ID de destino inmutable y estado anterior;
2. diferencia esperada y aprobación técnica;
3. identidad utilizada, endpoint y versión del módulo;
4. ID de respuesta/solicitud del sistema de destino;
5. estado vuelto a leer y prueba funcional dependiente;
6. regla de rollback o roll-forward a partir del Point of No Return.

La idempotencia es una propiedad de la operación y de su modelo de estado, no del lenguaje. `Set-*` puede ser idempotente; `Add-*` normalmente no lo es; una operación HTTP PUT puede repetirse, mientras que un POST crea un objeto nuevo con cada reintento. Antes de realizar un reintento automático, se aclaran la semántica de la API, la ID de solicitud y el estado del servidor.

## Paralelismo, trabajos y runspaces

PowerShell puede paralelizar el trabajo mediante Background Jobs, Thread Jobs, Remoting Jobs, `ForEach-Object -Parallel` o sus propios grupos de runspaces. Estos modelos difieren en aislamiento de procesos, serialización, costes de inicio y estado compartido. Los trabajos remotos y de proceso serializan objetos; Thread Jobs y runspaces comparten un proceso y pueden ser más rápidos, pero también menos aislados ([Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs), [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)).

El paralelismo óptimo está limitado por el sistema de destino:

- límite de tasa de API y Retry-After;
- máximo de shells y operaciones remotas por usuario;
- grupo de conexiones LDAP/base de datos;
- throttling de Exchange y coste de cmdlets;
- memoria por runspace y tamaño de objetos serializados;
- requisitos de orden y bloqueo en el destino;
- presupuesto de errores y capacidad de reinicio selectivo.

Más hilos no aumentan automáticamente el rendimiento. Un script de administración mide la longitud de la cola, la latencia, la tasa de éxito y el throttling, limita la concurrencia y utiliza jitter/backoff. Guarda un punto de control por objeto de destino, en lugar de repetir ciegamente todo el lote tras un fallo parcial.

## Módulos de producto y API HTTP

PowerShell es a menudo solo el lado cliente de un plano de control remoto. El cmdlet puede validar parámetros localmente y, a continuación, llamar a REST, WSMan, RPC, CIM o una API propietaria. El espacio de errores abarca entonces el motor local, el módulo, la obtención de tokens, DNS, proxy, [TLS](/kb/tls), versión de API, autorización, throttling y el servicio de producto.

El **Microsoft Graph PowerShell SDK** genera cmdlets a partir de los metadatos de la API Graph. Los permisos siguen siendo permisos de Graph; el nombre del cmdlet no elimina la semántica de Delegated frente a Application, el consentimiento ni las restricciones de recursos ([Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)). El módulo **ExchangeOnlineManagement** proporciona cmdlets específicos de Exchange y utiliza conexiones basadas en REST para numerosas operaciones; las interfaces de Exchange y Graph no son API intercambiables pese a sus solapamientos ([Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)).

Antes de una ejecución de producción se guardan:

| Nivel | Qué registrar |
|---|---|
| Cliente | motor/edición, proceso, sistema operativo, zona horaria |
| Código | repositorio, commit, artefacto firmado, configuración |
| Dependencia | nombre del módulo, versión, ruta, repositorio, hash |
| Identidad | ID de objeto/aplicación, tipo de autenticación, tenant/realm |
| Permiso | roles, scopes, consentimiento, endpoint/configuración de sesión |
| Solicitud | ID de destino, operación, ID de correlación/solicitud, momento |
| Resultado | estado, clase de error, reintento, estado modificado |
| Verificación | comprobación independiente de lectura o de registro |

Por ello, la búsqueda de errores comienza en el contexto de ejecución: edición, host, usuario, idioma, versión del módulo y ruta de acceso remoto. Solo después se investiga el producto de destino.

## Diagnóstico: primero el contexto de ejecución, después el producto de destino

Una incidencia de PowerShell se investiga según las dependencias reales:

1. **Llamada:** ¿qué proceso, host, usuario, directorio de trabajo, modo de perfil y NonInteractive?
2. **Resolución:** ¿qué comando y qué versión de módulo se utilizan realmente?
3. **Política local:** ¿modo de lenguaje, App Control, Execution Policy, almacén de confianza y proxy?
4. **Transporte:** ¿DNS, TCP, TLS, WSMan/SSH o HTTPS están accesibles?
5. **Autenticación:** ¿qué flujo, token/clave/certificado, hora y tenant?
6. **Autorización:** ¿qué rol local, capacidad JEA, scopes de API y roles de producto?
7. **Estado de destino:** ¿existe el objeto bajo la ID estable utilizada y en el ámbito correcto?
8. **Respuesta del servidor:** ¿código de error, ID de solicitud, throttling, registros de auditoría y de producto?

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) comprueba la accesibilidad TCP en Windows, [`nc`](https://man.openbsd.org/nc) en sistemas Unix. Esto no demuestra ni una autenticación TLS ni de aplicación correcta, pero separa un error de conexión de las capas superiores. [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) lee registros de eventos en Windows; [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) lee el journal de systemd.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Transport- und Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mail01.example.ch -Port 5986 -InformationLevel Detailed
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' -MaxEvents 100 |
  Select-Object TimeCreated,Id,LevelDisplayName,Message
Get-Command Invoke-Command -All |
  Select-Object CommandType,Source,Version,Definition</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz mail01.example.ch 22
journalctl --since '-30 min' --no-pager -u sshd
pwsh -NoProfile -Command '
  Get-Command Invoke-Command -All |
    Select-Object CommandType,Source,Version,Definition
'</code></pre>
  </div>
</div>

## Registro, transcripciones y protección de datos

PowerShell puede utilizar Script Block Logging, Module Logging, Protected Event Logging y Transcription. En Windows, los eventos de bloques de script aparecen en el canal `Microsoft-Windows-PowerShell/Operational`. [`Start-Transcript`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript) registra la representación textual de una sesión; no es un registro de auditoría estructurado completo y no captura toda actividad fuera del host ([Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)).

El registro aumenta la visibilidad y, al mismo tiempo, el riesgo de almacenar tokens, contraseñas, contenidos de correo o datos personales. Microsoft recomienda Protected Event Logging para registros sensibles de bloques de script. Además, se definen la retención, el acceso, el reenvío centralizado, la sincronización horaria y el almacenamiento protegido contra manipulaciones ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

Para automatizaciones propias, los eventos estructurados son más adecuados que los textos de consola con formato libre. Un evento contiene como mínimo hora en UTC, ID de ejecución, versión de código, ID de destino, acción, resultado, duración, ID de solicitud y una causa de error clasificada. Los secretos y payloads innecesarios se eliminan antes de registrar.

## Control de calidad y despliegue

[`Invoke-ScriptAnalyzer`](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) comprueba código de PowerShell mediante reglas estáticas. Esto detecta muchos problemas de estilo, compatibilidad y seguridad, pero no sustituye ni las pruebas ni una integración de producto. Una canalización fiable incluye:

- comprobación del analizador y de reglas estáticas;
- pruebas unitarias para funciones puras y mapeo de datos;
- pruebas de contrato frente a esquemas API y respuestas simuladas;
- prueba de integración con los permisos mínimos;
- salida de `-WhatIf`/plan cuando sea técnicamente posible;
- prueba de conjuntos vacíos, duplicados, paginación, throttling y errores parciales;
- firma, hash y artefacto inmutable;
- despliegue gradual con métricas y criterios de interrupción;
- verificación independiente de lectura en el sistema de destino.

Los perfiles se excluyen normalmente de la automatización de producción con `-NoProfile`. Los procesos no interactivos utilizan `-NonInteractive`, codificaciones definidas, rutas explícitas y una zona horaria fija. El despliegue entrega módulos junto con el software o utiliza un repositorio interno controlado; no instala sin más la dependencia más reciente en cada ejecución.

## Historia técnica

Jeffrey Snover formuló en el **Monad Manifesto** un modelo de administración unificado y orientado a objetos: herramientas pequeñas, nomenclatura coherente, funcionalidad común de analizador y canalización, así como interfaces de administración mediante .NET. Microsoft describe el documento como el origen del desarrollo que más tarde se convertiría en PowerShell ([PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)).

En abril de 2006, Microsoft cambió el nombre del proyecto desarrollado bajo el nombre en clave Monad a **Windows PowerShell**; la primera línea de producto estaba estrechamente ligada a Windows y .NET Framework ([Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)). En esta línea surgieron las convenciones verbo-sustantivo de cmdlets, la canalización de objetos, los proveedores, el acceso remoto, los módulos, los flujos de trabajo y Desired State Configuration.

En 2016, Microsoft publicó PowerShell bajo licencia MIT y desarrolló una línea multiplataforma basada en .NET Core. PowerShell Core 6 incorporó compatibilidad con Linux/macOS y acceso remoto SSH; PowerShell 7 continuó el nombre del producto sin «Core». La base de código y el desarrollo de PowerShell 7+ están en el repositorio público; los cambios no se retroportan a Windows PowerShell 5.1 ([PowerShell GitHub repository](https://github.com/PowerShell/PowerShell), [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)).

Esta historia explica muchas limitaciones operativas actuales: dos ediciones en Windows, módulos con dependencias de .NET Framework, distintos transportes de acceso remoto y una combinación de interfaces locales de administración de Windows y API HTTP multiplataforma. Por ello, para los administradores PowerShell es menos «un número de versión» que una combinación concreta de motor, tiempo de ejecución, módulo, transporte y producto de destino.

## Lo que un administrador debe registrar de un vistazo

Un script solo es operable cuando, además del código, se conocen el tiempo de ejecución, los módulos, la identidad, el sistema de destino y el tratamiento de errores. La tabla resume la evidencia mínima necesaria para ello.

| Pregunta | Evidencia mínima |
|---|---|
| ¿Qué PowerShell se ejecuta? | ruta del proceso, PSEdition, PSVersion, .NET, SO/arquitectura |
| ¿Qué se ejecuta? | CommandType, Source, Version y Definition resueltos |
| ¿En qué contexto? | host, runspace, modo de perfil, directorio de trabajo, modo de lenguaje |
| ¿Con qué dependencias? | ruta de módulos, manifiesto, versión, repositorio, hash |
| ¿Contra qué destino? | endpoint, tenant/realm, ID de objeto estable, versión de API/sesión |
| ¿Con qué identidad? | usuario/aplicación/identidad administrada, flujo de autenticación, ID de certificado/clave |
| ¿Con qué permisos? | rol local, endpoint JEA, scopes, RBAC de producto |
| ¿Qué debe cambiar? | diferencia actual/deseada, plan, ámbito, aprobación |
| ¿Qué cambió realmente? | respuesta del servidor, ID de solicitud, lectura posterior y prueba funcional |
| ¿Cómo se trata un fallo parcial? | ErrorAction, código de salida, reintento, punto de control, compensación |
| ¿Cómo se audita? | ID de ejecución, registros estructurados, Transcript/Operational Log, retención |
| ¿Cómo se repite o revierte? | clave de idempotencia, Point of No Return, roll-forward/-back |

## Fuentes

- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)
- [Microsoft – PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary)
- [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)
- [Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)
- [Microsoft – about_Automatic_Variables](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables)
- [Microsoft – Get-Host](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host)
- [Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)
- [Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)
- [Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)
- [Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects)
- [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)
- [Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)
- [Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)
- [Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)
- [Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)
- [Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements)
- [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)
- [Microsoft – Test-WSMan](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman)
- [Microsoft – New-PSSession](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession)
- [Microsoft – Invoke-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command)
- [OpenBSD – ssh](https://man.openbsd.org/ssh)
- [Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)
- [Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)
- [Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)
- [Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)
- [Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)
- [Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)
- [Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling)
- [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)
- [Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess)
- [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)
- [Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs)
- [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)
- [Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
- [Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft – Start-Transcript](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript)
- [Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)
- [Microsoft – PSScriptAnalyzer overview](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview)
- [PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)
- [Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)
- [PowerShell – GitHub repository](https://github.com/PowerShell/PowerShell)
- [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)
