---
title: "Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de Wrapper y publicada la v2"
navTitle: "Exchange SU 09/2026"
description: "La SU de septiembre corrige nueve vulnerabilidades en Exchange SE y 2019 (ocho en Exchange 2016), incluida una vulnerabilidad de suplantación con CVSS 9.3, y soluciona el problema de Wrapper en entornos híbridos. El 2 de octubre se publicó una v2 con una CVE adicional; además, hay tres problemas conocidos con soluciones alternativas y un SettingOverride que ahora debe eliminarse."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "11 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
translationSourceHash: 1f9a8027af593738b429df5a1b20adab97301c20761ef0cfe02e2691a12e880a
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:46:42.801Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema
---

# Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de Wrapper y publicada la v2

Microsoft publicó actualizaciones de seguridad (SU) para Exchange el 8 de septiembre de 2026. Corrigen nueve vulnerabilidades en Exchange SE y Exchange 2019, y ocho en Exchange 2016. Ninguna era conocida públicamente de antemano, ninguna se está explotando activamente según la Security Update Guide, y Microsoft las clasifica todas como *Important* con «Exploitation Less Likely». Sin embargo, el valor CVSS más alto, 9.3, supera claramente el del mes anterior. Este mes es particular por tres razones: la SU corrige el problema de los *mensajes Wrapper* en buzones compartidos, abierto desde junio; introduce tres problemas conocidos nuevos o persistentes; y el 2 de octubre Microsoft publicó una **versión 2** que corrige una vulnerabilidad adicional.

## Para qué versiones de Exchange está disponible la actualización

Las SU del 8 de septiembre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, compilación 15.2.2562.49; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5121609, compilación 15.2.1748.51; solo mediante el **programa ESU de periodo 2**.
- **Exchange Server 2019 CU14**: KB5121610, compilación 15.2.1544.46; solo mediante ESU de periodo 2.
- **Exchange Server 2016 CU23**: KB5121611, compilación 15.1.2507.73; solo mediante ESU de periodo 2.

Exchange 2016 y 2019 están fuera de soporte. Según Microsoft, las SU de mayo a octubre de 2026 solo las reciben organizaciones inscritas en el programa ESU de periodo 2. Según los artículos de KB, esta elegibilidad se extiende hasta octubre de 2026. Además, existe presión desde Exchange Online: desde la segunda semana de septiembre, Exchange Online limita y bloquea el flujo de correo híbrido de servidores con una versión anterior a octubre de 2025; los detalles están en el [artículo sobre la aplicación del transporte](/blog/exchange-online-transport-enforcement-hybrid-server). Según el anuncio, Exchange Online ya está protegido; aun así, en entornos híbridos cada servidor de Exchange necesita la SU, al igual que las máquinas con las Exchange Management Tools.

Puede comparar su versión actual con el resumen de [números de compilación de Exchange](/tools/exchange-builds).

## Resumen de las vulnerabilidades

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Suplantación (Cross-Site-Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Suplantación (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Manipulación | 6.5 |
| CVE-2026-69382 | Divulgación de información | 5.9 |

CVE-2026-55007 no afecta a Exchange 2016; la Security Update Guide solo enumera Exchange SE y 2019 CU14/CU15 para esta CVE. Un detalle de la documentación: CVE-2026-69380 no figura en la lista de CVE de los artículos de KB de las cuatro SU de septiembre. Sin embargo, la Security Update Guide indica precisamente las cuatro compilaciones de septiembre como corrección para esta CVE (a fecha de 7 de octubre de 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** tiene el valor más alto, CVSS 9.3. Según Microsoft, un atacante no autenticado puede enviar una invitación de calendario manipulada con un enlace de reunión malicioso; si la destinataria o el destinatario abre la reunión y selecciona el enlace para unirse, se activa el Cross-Site-Scripting. Por tanto, se requiere interacción del usuario, pero no una cuenta en la organización.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) solo requiere una cuenta con pocos permisos y un buzón asignado. Según las preguntas frecuentes de la Security Update Guide, un atacante puede hacerse pasar por otro usuario mediante debilidades en la validación de solicitudes y tokens de identidad, y tomar control de los buzones de todos los usuarios de Exchange: leer y enviar correos, así como descargar adjuntos. Una única cuenta de usuario comprometida basta como punto de partida.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) conduce al mismo resultado, la toma de control de todos los buzones, pero requiere pertenecer a un grupo de roles con privilegios elevados.

La situación inicial de las dos vulnerabilidades de Remote Code Execution es diferente: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requiere una cuenta autenticada con pocos permisos; [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) puede desencadenarse sin inicio de sesión mediante un adjunto de Visio manipulado, pero según Microsoft requiere poca memoria disponible de forma persistente en el sistema de destino. Las otras cuatro vulnerabilidades son: CVE-2026-69378 (DoS mediante recursión no controlada, sin inicio de sesión), CVE-2026-69361 (SSRF, el servidor envía solicitudes HTTP a sistemas internos o de loopback), CVE-2026-69375 (un atacante autenticado puede sustituir el contenido de archivos) y CVE-2026-69382 (divulgación de credenciales mediante un algoritmo criptográfico débil; requiere una cookie de autenticación previamente obtenida).

## Versión 2 del 2 de octubre: se añade CVE-2026-96940

El 2 de octubre de 2026, Microsoft publicó la «versión 2» de las SU de septiembre. Según el anuncio, la única diferencia respecto a la primera versión es la corrección adicional para **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Esta vulnerabilidad de Elevation of Privilege tiene un CVSS de 8.8, no es conocida públicamente ni está siendo explotada, pero Microsoft es la única de las diez CVE que la califica como **«Exploitation More Likely»**. Un atacante autenticado puede acceder a buzones ajenos de la misma organización y leer correos con sus adjuntos. Exchange Online ya está corregido del lado del servidor.

| Versión | KB | Compilación v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En la práctica, esto significa que la corrección para CVE-2026-96940 solo está incluida en las compilaciones v2. Los servidores que ya ejecutan la SU del 8 de septiembre también necesitan v2. Quien vaya a aplicar los parches ahora puede instalar directamente v2, ya que las SU son acumulativas. Los artículos de KB no indican expresamente si los servidores con la primera SU de septiembre deben instalar obligatoriamente v2; dado que la nueva CVE solo se corrige con v2, esa es la interpretación lógica.

## Problema de Wrapper corregido: eliminar ahora el SettingOverride

El problema conocido desde la SU de junio, por el que aparecían *mensajes Wrapper* en la bandeja de entrada de buzones compartidos en entornos híbridos, se ha corregido con la SU de septiembre en las cuatro versiones. En el [artículo de agosto](/blog/exchange-security-updates-august-2026) todavía se indicaba que el SettingOverride documentado como solución alternativa podía permanecer. Tras instalar la SU de septiembre ocurre lo contrario: Microsoft recomienda en el artículo de soporte correspondiente comprobar y eliminar el Override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Comando | Efecto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Comprueba si el Override de solución alternativa está configurado en la organización. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Elimina el Override una vez instalada la SU de septiembre. |

</details>

Si el primer comando indica que no se encontró el objeto `DisableBlockSharedAndUserMailboxHeaders`, según Microsoft no hay que hacer nada más.

Para Exchange SE, la SU también corrige un error en las consultas híbridas de disponibilidad mediante Microsoft Graph: los usuarios locales veían los intervalos ocupados de los buzones de Exchange Online desplazados por su propia diferencia UTC, sin mensaje de error.

## Problemas conocidos

**Los calendarios publicados (.ics) devuelven HTTP 500 a aplicaciones de calendario.** El problema existe desde la SU de agosto (Exchange SE desde la compilación 15.2.2562.46, así como Exchange 2019 y 2016), y no se ha corregido ni en la SU de septiembre ni en v2. Las suscripciones a calendarios publicados anónimamente ya no se actualizan; la misma URL funciona en el navegador. Según Microsoft, la causa es que Exchange identifica a los clientes mediante el User-Agent; las aplicaciones de calendario sin identificador de navegador terminan en una ruta de código que la SU de agosto desactivó. Como solución alternativa, Microsoft describe una regla de reescritura de URL en el sitio «Exchange Back End» de IIS, que añade el parámetro `layout=premium` a las solicitudes .ics bajo `/owa/calendar/`; para ello debe estar instalado el módulo IIS URL Rewrite. Los pasos exactos (mediante el Administrador de IIS o directamente en `applicationHost.config`) se encuentran en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft no indica una fecha para la corrección (a fecha de 7 de octubre de 2026).

**Disponibilidad para buzones delegados en entornos híbridos (solo Exchange SE).** Si la consulta de disponibilidad se configura exclusivamente a través de Graph API, las consultas de buzones de Exchange Online mediante acceso local delegado fallan. Outlook muestra «Your server location could not be determined», OWA muestra «No information» y en los registros de EWS aparece `(403) Forbidden`. La solución alternativa documentada vuelve a dirigir las consultas mediante EWS en lugar de Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | El Override que controla el redireccionamiento de las consultas de disponibilidad mediante Microsoft Graph. |
| `-Parameters "Enabled=False"` | Desactiva la ruta Graph; las consultas vuelven a realizarse mediante EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige la llamada de diagnóstico al servicio de topología. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga la Variant Configuration para que el Override se aplique sin esperar. |

</details>

En el anuncio de v2, Microsoft enumera este problema entre los problemas corregidos. Quien haya configurado la solución alternativa debería comprobar en el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) tras instalar v2 si debe revertirla; a fecha de 7 de octubre de 2026 aún no había instrucciones para ello.

**Interbloqueo de ContentEngine debido a la ausencia de archivos de reglas WordBreaker coreanos (solo Exchange SE).** Con la SU de septiembre (compilación 15.2.2562.49 y compilación v2 15.2.2562.53), no se instalan los archivos de reglas para el WordBreaker coreano actualizado. Las consecuencias son: resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI que se bloquean o pierden la conexión. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) identifica como afectados los entornos que procesan correos con contenido en coreano. Esto no depende del idioma de sus propios usuarios: los mensajes con texto coreano llegan también a organizaciones sin correspondencia en coreano, por ejemplo como spam o boletines. Por tanto, los entornos suizos no quedan excluidos de antemano. Microsoft sigue investigando el problema; la comprobación y la solución alternativa se explican en la siguiente sección.

## Solución alternativa para el interbloqueo de WordBreaker

La solución alternativa de KB5130098 añade dos archivos de reglas del paquete SQL Server 2025 Express RTM. Microsoft la somete a condiciones: no debe aplicarse solo por los síntomas, sino únicamente en servidores que cumplan todos los criterios de comprobación y a los que les falten **ambos** archivos. El [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) no comprueba estos archivos; la versión v26.10.02 del 2 de octubre de 2026 no contiene una comprobación correspondiente (a fecha de 9 de octubre de 2026). Por ello, la comprobación debe realizarse manualmente.

### 1. Comprobar el servidor

El siguiente script no modifica nada en el sistema. Lee la ruta de instalación de la variable de entorno `ExchangeInstallPath` y muestra los valores que KB5130098 indica como requisito.

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$exsetup = Get-Item (Join-Path $env:ExchangeInstallPath 'Bin\ExSetup.exe')
$dll     = Get-Item (Join-Path $native 'korwbrkr.dll')
[pscustomobject]@{
    ExSetup         = $exsetup.VersionInfo.FileVersion
    KorwbrkrVersion = $dll.VersionInfo.FileVersion
    KorwbrkrBytes   = $dll.Length
    KorwbrkrSha256  = (Get-FileHash -LiteralPath $dll.FullName -Algorithm SHA256).Hash
    KoTokenRule     = Test-Path (Join-Path $native 'ko.token.rule.bin')
    KoComplexRule   = Test-Path (Join-Path $native 'ko.complex.rule.bin')
}
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Línea | Efecto |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Crea la ruta al directorio donde deben ubicarse los archivos de reglas. |
| `Get-Item ...\ExSetup.exe` | Lee el archivo cuya versión indica la compilación de Exchange instalada. |
| `VersionInfo.FileVersion` | Versión del archivo a partir de los recursos del EXE o DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcula el hash SHA256 de la `korwbrkr.dll` instalada. |
| `Test-Path ...` | Devuelve `True` si el archivo de reglas correspondiente ya existe. |

</details>

La solución alternativa solo está permitida si todos los valores coinciden:

| Valor | Esperado |
| --- | --- |
| `ExSetup` | 15.02.2562.049 o 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | ambos `False` |

Si la compilación, la versión o el hash de la DLL difieren, Microsoft remite al soporte. Si uno de los dos archivos de reglas ya existe, deténgase y no sobrescriba nada.

### 2. Obtener los archivos de reglas

Los archivos proceden de `SQLEXPR_x64_ENU.exe`, el paquete x64 en inglés de SQL Server 2025 Express RTM (versión 17.0.1000.7, 748772024 bytes). Microsoft recomienda realizar este paso en una estación de trabajo de administración, no en el servidor Exchange. SQL Server no se instala: el paquete solo se descomprime y el MSI Full-Text se extrae únicamente de forma administrativa. Primero compruebe la firma y el hash de la descarga:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Se espera el estado `Valid` con un certificado de Microsoft y el valor SHA256 `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. A continuación, descomprima y extraiga:

```powershell
$work = 'C:\Temp\KoreanRules'
$p = Start-Process .\SQLEXPR_x64_ENU.exe -ArgumentList '/q', "/x:$work\Media" -Wait -PassThru
$p.ExitCode
$msi = "$work\Media\x64\Setup\SQL_FULLTEXT.MSI"
$msiArgs = '/a', $msi, "TARGETDIR=$work\Files", '/qn', '/L*V', "$work\extract.log"
$p = Start-Process msiexec.exe -ArgumentList $msiArgs -Wait -PassThru
$p.ExitCode
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `/q` | Descomprime sin diálogos. |
| `/x:<Pfad>` | Modo de extracción del paquete SQL: descomprime los medios de instalación en la carpeta sin iniciar SQL Setup. |
| `-Wait -PassThru` | Espera a que termine el proceso y devuelve el objeto de proceso para poder leer el código de salida. |
| `msiexec /a <MSI>` | Instalación administrativa: deja el contenido del MSI como archivos, sin instalar ni registrar nada. |
| `TARGETDIR=<Pfad>` | Carpeta de destino para los archivos extraídos. |
| `/qn` | Sin interfaz de usuario. |
| `/L*V <Datei>` | Registro detallado que muestra la causa en caso de error. |

</details>

Ambos códigos de salida deben ser `0`. Si la extracción falla, guarde `extract.log` y deténgase. A continuación, compruebe el tamaño y el hash de los dos archivos:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Archivo | Bytes | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Los archivos se encuentran en el árbol extraído bajo `...\MSSQL\Binn\ftcomponents\wordbreakers`. Transfiera únicamente estos dos archivos a una carpeta de staging del servidor Exchange; no transfiera medios de SQL y, expresamente, tampoco la `korwbrkr.dll` del paquete SQL.

### 3. Copiar archivos

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copia el archivo desde la carpeta de staging; la ruta se evalúa sin comodines. |
| `-Destination $native` | Directorio de destino junto a la `korwbrkr.dll` instalada. |
| `Get-FileHash ... -Algorithm SHA256` | Comprueba de nuevo el hash en la ubicación de destino respecto a la tabla anterior. |

</details>

Utilice `Copy-Item`, no `Move-Item`. Un archivo copiado recibe los permisos heredados del directorio de destino; uno movido conserva los permisos de su carpeta de origen. Los archivos necesitan los permisos de lectura normales de `Native`. No sustituya otros archivos de este directorio.

### 4. Reiniciar Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

El reinicio interrumpe temporalmente la búsqueda y el procesamiento de contenido en el servidor; Microsoft prevé para ello una ventana de mantenimiento. No es necesario reiniciar el propio servidor. No finalice procesos de Exchange por la fuerza: si el servicio no inicia correctamente, según Microsoft el siguiente paso es contactar con el soporte.

### 5. Comprobación

Host Controller inicia por sí mismo el proceso `NodeRunner.exe` de ContentEngine; no lo inicie manualmente. Esta consulta muestra si se está ejecutando:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Lee todos los procesos NodeRunner junto con la línea de comandos. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtra el proceso de ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Muestra el ID de proceso y la hora de inicio; la hora de inicio debe ser posterior al reinicio del servicio. |

</details>

Si no aparece ningún proceso o termina repetidamente, no procese más servidores. De lo contrario, compruebe en Outlook en la Web, con un buzón cuya copia de base de datos activa se encuentre en este servidor, si se entregan y encuentran correos nuevos, incluidos correos con texto coreano. Compruebe también el servicio que se vio afectado originalmente (entrega, conexión de Outlook). Supervise por separado el retraso en el índice de búsqueda: que se encuentren correos nuevos no demuestra que ya se hayan procesado elementos más antiguos. Procese otros servidores uno por uno y solo después de comprobar correctamente el anterior.

## Instalación y tareas posteriores

Microsoft recomienda el procedimiento habitual: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la versión está desactualizada, instalar la SU, reiniciar el servidor y comprobar que todos los servicios de Exchange estén en ejecución. Posteriormente, el Health Checker también muestra si la SU se instaló correctamente. La Security Update Guide indica que estas actualizaciones requieren reinicio.

Después de la instalación hay tres tareas posteriores:

1. Eliminar el Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` si está configurado (véase arriba).

2. En Exchange SE, usar el script de comprobación de la [sección WordBreaker](#workaround-für-den-wordbreaker-deadlock) para verificar si faltan los dos archivos de reglas, y comprobar si se produce el problema de disponibilidad. Aplique las soluciones alternativas solo si se cumplen las condiciones respectivas.

3. Para calendarios publicados con suscriptores externos, configurar la regla de reescritura de URL de KB5126672 si no se hizo ya desde la SU de agosto.

Además, desde julio sigue pendiente comprobar si la mitigación CVE-2026-42897 (M2.1.0) continúa activa; cómo eliminarla se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

Instale directamente las compilaciones v2 del 2 de octubre en todos los servidores Exchange y las máquinas con Management Tools; los servidores con la SU del 8 de septiembre necesitan adicionalmente v2 para CVE-2026-96940. La vulnerabilidad de suplantación con CVSS 9.3 y la toma de control de buzones mediante una cuenta de usuario con pocos permisos (CVE-2026-69380) son motivo suficiente para no esperar al próximo Patch Tuesday. Después, elimine el Override de Wrapper, compruebe los tres problemas conocidos y ejecute el Health Checker. El Health Checker no informa de los archivos WordBreaker ausentes; para ello se necesita el script de comprobación de la sección WordBreaker. El programa ESU para Exchange 2016 y 2019 termina en octubre de 2026; la migración a Exchange SE ya no puede aplazarse más.

## Fuentes

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Anuncio oficial de lanzamiento con versiones compatibles, indicación sobre ESU, problemas conocidos, problemas corregidos y procedimiento de instalación (consultado mediante el feed RSS del Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de la v2 del 2 de octubre de 2026; la única diferencia es CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Lista de CVE, problemas corregidos y los tres problemas conocidos para Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Artículo de KB para Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Artículo de KB para Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Artículo de KB para Exchange 2016 CU23, sin CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, evaluación de explotación y preguntas frecuentes sobre las nueve CVE de septiembre y CVE-2026-96940, incluidas las compilaciones afectadas por cada CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Artículo de KB sobre la v2 para Exchange SE; los artículos v2 para 2019 y 2016 son KB5129956 a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de la v2 del 2 de octubre de 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Corrección en la SU de septiembre e instrucciones para eliminar el SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa y solución alternativa de reescritura de URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas y solución alternativa mediante SettingOverride para Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, criterios de comprobación, tamaños y hashes de archivo esperados, obtención desde SQL Server 2025 Express, reinicio de servicio y comprobación.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Error de zona horaria en Exchange SE corregido con la SU de septiembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Desglose en alemán de las nueve CVE con valores CVSS y compilaciones.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Resumen en alemán con referencias a la posterior actualización sustitutiva.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Código fuente del Exchange Health Checker; versión v26.10.02 del 2 de octubre de 2026 sin comprobación de los archivos WordBreaker (a fecha de 9 de octubre de 2026).
