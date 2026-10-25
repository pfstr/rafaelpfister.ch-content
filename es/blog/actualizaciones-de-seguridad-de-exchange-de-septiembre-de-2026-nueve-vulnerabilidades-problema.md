---
title: "Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de los wrappers y publicada v2"
navTitle: "Exchange SU 09/2026"
description: "La SU de septiembre corrige nueve vulnerabilidades en Exchange SE y 2019 (ocho en Exchange 2016), incluida una vulnerabilidad de suplantación con CVSS 9.3, y soluciona el problema de los wrappers en entornos híbridos. El 2 de octubre se publicó una v2 con una CVE adicional; también hay tres problemas conocidos con soluciones alternativas y un SettingOverride que ahora debe eliminarse."
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
translationSourceHash: fb8f756123ccec8c1983887594f47b521485f9fcf463783b480bd5eaaac8a427
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:40:59.205Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema
---

# Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de los wrappers y publicada v2

Microsoft publicó el 8 de septiembre de 2026 actualizaciones de seguridad (SU) para Exchange Server. Corrigen nueve vulnerabilidades en Exchange SE y Exchange 2019, y ocho en Exchange 2016. Ninguna era conocida públicamente de antemano, ninguna está siendo explotada activamente según la Security Update Guide y Microsoft clasifica todas como *Important* con «Exploitation Less Likely». Sin embargo, el valor CVSS más alto, 9.3, es considerablemente superior al del mes anterior. Este mes es especial por tres motivos: la SU soluciona el problema de los *mensajes wrapper* en buzones compartidos, abierto desde junio; incorpora tres problemas conocidos nuevos o persistentes; y el 2 de octubre Microsoft publicó una **versión 2** que corrige una vulnerabilidad adicional.

## Para qué versiones de Exchange está disponible la actualización

Las SU del 8 de septiembre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, compilación 15.2.2562.49; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5121609, compilación 15.2.1748.51; solo mediante el **programa ESU de período 2**.
- **Exchange Server 2019 CU14**: KB5121610, compilación 15.2.1544.46; solo mediante ESU de período 2.
- **Exchange Server 2016 CU23**: KB5121611, compilación 15.1.2507.73; solo mediante ESU de período 2.

Exchange 2016 y 2019 ya no tienen soporte. Según Microsoft, las SU de mayo a octubre de 2026 solo las reciben las organizaciones inscritas en el programa ESU de período 2. Según los artículos de KB, esta elegibilidad se extiende hasta octubre de 2026. También existe presión desde Exchange Online: desde la segunda semana de septiembre, Exchange Online limita y bloquea el flujo de correo híbrido de servidores con un nivel anterior a octubre de 2025; los detalles están en el [artículo sobre la aplicación de transporte](/blog/exchange-online-transport-enforcement-hybrid-server). Según el anuncio, Exchange Online ya está protegido; no obstante, en entornos híbridos todos los servidores Exchange requieren la SU, al igual que las máquinas con Exchange Management Tools.

Puede comparar su versión actual con la lista de [números de compilación de Exchange](/tools/exchange-builds).

## Resumen de las vulnerabilidades

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Suplantación (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevación de privilegios | 9.1 |
| CVE-2026-69355 | Ejecución remota de código | 8.8 |
| CVE-2026-55007 | Ejecución remota de código | 8.1 |
| CVE-2026-69380 | Elevación de privilegios | 8.1 |
| CVE-2026-69378 | Denegación de servicio | 7.5 |
| CVE-2026-69361 | Suplantación (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Manipulación | 6.5 |
| CVE-2026-69382 | Divulgación de información | 5.9 |

CVE-2026-55007 no afecta a Exchange 2016; la Security Update Guide solo enumera para ella Exchange SE y 2019 CU14/CU15. Un detalle sobre la documentación: CVE-2026-69380 falta en la lista de CVE de los artículos de KB de las cuatro SU de septiembre. Sin embargo, la Security Update Guide indica precisamente las cuatro compilaciones de septiembre como corrección para esta CVE (a 7 de octubre de 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356) tiene el valor más alto, CVSS 9.3. Según Microsoft, un atacante no autenticado puede enviar una invitación de calendario manipulada con un enlace de reunión malicioso; si la destinataria o el destinatario abre la reunión y selecciona el enlace para unirse, se desencadena el Cross-Site Scripting. Por tanto, requiere interacción del usuario, pero no una cuenta en la organización.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (elevación de privilegios, CVSS 8.1) solo requiere una cuenta con pocos privilegios y un buzón asignado. Según las FAQ de la Security Update Guide, un atacante puede, mediante debilidades en la comprobación de solicitudes y tokens de identidad, hacerse pasar por otro usuario y tomar el control de los buzones de todos los usuarios de Exchange: leer y enviar correos, y descargar adjuntos. Basta una única cuenta de usuario comprometida como punto de partida.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (elevación de privilegios, CVSS 9.1) conduce al mismo resultado, la toma de control de todos los buzones, pero requiere pertenecer a un grupo de roles con privilegios elevados.

La situación inicial de las dos vulnerabilidades de ejecución remota de código es distinta: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requiere una cuenta autenticada con pocos privilegios; [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) puede desencadenarse sin iniciar sesión mediante un adjunto de Visio manipulado, pero según Microsoft requiere memoria de trabajo persistentemente escasa en el sistema objetivo. Las otras cuatro vulnerabilidades son: CVE-2026-69378 (DoS por recursión no controlada, sin inicio de sesión), CVE-2026-69361 (SSRF, el servidor envía solicitudes HTTP a sistemas internos o de loopback), CVE-2026-69375 (un atacante autenticado puede reemplazar contenidos de archivos) y CVE-2026-69382 (divulgación de credenciales mediante un algoritmo criptográfico débil, requiere una cookie de autenticación ya obtenida).

## Versión 2 del 2 de octubre: se añade CVE-2026-96940

El 2 de octubre de 2026 Microsoft publicó la «versión 2» de las SU de septiembre. Según el anuncio, la única diferencia con la primera versión es la corrección adicional para **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Esta vulnerabilidad de elevación de privilegios tiene CVSS 8.8, no es conocida públicamente ni está siendo explotada, pero Microsoft la valora como la única de las diez CVE con **«Exploitation More Likely»**. Un atacante autenticado puede acceder así a buzones ajenos de la misma organización y leer correos con sus adjuntos. Exchange Online ya está corregido en el lado del servidor.

| Versión | KB | Compilación v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En la práctica, esto significa que la corrección de CVE-2026-96940 solo está incluida en las compilaciones v2. Los servidores que ya ejecutan la SU del 8 de septiembre necesitan además v2. Quien aplique parches ahora puede instalar directamente v2, ya que las SU son acumulativas. Los artículos de KB no dicen expresamente si los servidores con la primera SU de septiembre deben instalar v2 obligatoriamente; dado que la nueva CVE solo se corrige con v2, esa es la interpretación lógica.

## Corregido el problema de los wrappers: elimine ahora SettingOverride

El problema conocido desde la SU de junio, por el que aparecen *mensajes wrapper* en la bandeja de entrada de buzones compartidos en entornos híbridos, se corrige con la SU de septiembre en las cuatro versiones. El [artículo de agosto](/blog/exchange-security-updates-august-2026) todavía indicaba que podía mantenerse el SettingOverride documentado como solución alternativa. Tras instalar la SU de septiembre ocurre lo contrario: Microsoft recomienda en el artículo de soporte correspondiente comprobar y eliminar el override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Comando | Efecto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Comprueba si el override de la solución alternativa está configurado en la organización. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Elimina el override una vez instalada la SU de septiembre. |

</details>

Si el primer comando informa de que no se encontró el objeto `DisableBlockSharedAndUserMailboxHeaders`, según Microsoft no es necesario hacer nada más.

Para Exchange SE, la SU también corrige un error en las consultas híbridas de libre/ocupado mediante Microsoft Graph: los usuarios locales veían las horas ocupadas de buzones de Exchange Online desplazadas según su propio desfase UTC, sin mensaje de error.

## Problemas conocidos

**Los calendarios publicados (.ics) devuelven HTTP 500 a aplicaciones de calendario.** El problema existe desde la SU de agosto (Exchange SE desde la compilación 15.2.2562.46, así como Exchange 2019 y 2016) y tampoco se corrige en la SU de septiembre ni en v2. Las suscripciones a calendarios publicados de forma anónima ya no se actualizan; la misma URL funciona en el navegador. Según Microsoft, la causa es que Exchange identifica a los clientes por el agente de usuario: las aplicaciones de calendario sin identificador de navegador llegan a una ruta de código que la SU de agosto desactivó. Como solución alternativa, Microsoft describe una regla de reescritura de URL en el sitio «Exchange Back End» de IIS, que añade el parámetro `layout=premium` a las solicitudes .ics bajo `/owa/calendar/`; para ello debe estar instalado el módulo IIS URL Rewrite. Los pasos exactos (mediante IIS Manager o directamente en `applicationHost.config`) están en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Desde el 13 de octubre de 2026, según KB5126672, el problema se corrige con las SU de octubre (KB5128895 para Exchange SE, KB5128896 a KB5128898 para 2019 y 2016); tras instalarlas debe eliminarse de nuevo la regla de reescritura.

**Libre/ocupado para buzones delegados en entornos híbridos (solo Exchange SE).** Si la consulta de disponibilidad se configura exclusivamente mediante la API Graph, las consultas para buzones de Exchange Online mediante acceso local delegado fallan. Outlook muestra «Your server location could not be determined», OWA muestra «No information» y en los registros EWS aparece `(403) Forbidden`. La solución alternativa documentada vuelve a enrutar las consultas mediante EWS en lugar de Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | El override que controla el enrutamiento de las consultas de disponibilidad mediante Microsoft Graph. |
| `-Parameters "Enabled=False"` | Desactiva la ruta Graph; las consultas vuelven a pasar por EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige la llamada de diagnóstico al servicio de topología. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga la Variant Configuration para que el override tenga efecto sin esperar. |

</details>

Microsoft enumera este problema como corregido en el anuncio de v2. Mientras tanto, el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) informa de que el problema se ha corregido en el servicio. Si configuró la solución alternativa, reviértala según KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Muestra si el override está configurado y qué valor tiene. |
| `-Parameters "Enabled=True"` | Vuelve a activar la ruta Graph para las consultas de disponibilidad. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga la Variant Configuration para que el cambio tenga efecto sin esperar. |

</details>

**Interbloqueo de ContentEngine debido a archivos de reglas WordBreaker coreanos ausentes (solo Exchange SE).** Con la SU de septiembre (compilación 15.2.2562.49 y compilación v2 15.2.2562.53), no se instalan los archivos de reglas para el WordBreaker coreano actualizado. Las consecuencias son resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI que se bloquean o pierden la conexión. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) indica como afectados los entornos que procesan correos con contenido en coreano. Esto no depende del idioma de sus propios usuarios: los mensajes con texto coreano también llegan a organizaciones sin correspondencia en coreano, por ejemplo como spam o boletines. Por ello, los entornos suizos no están excluidos de entrada. Desde el 13 de octubre de 2026, según KB5130098, el problema se corrige con KB5128895 (compilación 15.2.2562.56) o una actualización posterior; según Microsoft, después no se requieren más pasos, tampoco en servidores donde se haya aplicado la solución manual. La comprobación y la solución alternativa de la sección siguiente solo afectan ya a servidores con la SU de septiembre o v2.

## Solución alternativa para el interbloqueo de WordBreaker

La solución alternativa de KB5130098 añade dos archivos de reglas desde el paquete SQL Server 2025 Express RTM. Microsoft la vincula a condiciones: no debe aplicarse solo por los síntomas, sino únicamente en servidores que cumplan todos los criterios de comprobación y a los que les falten **ambos** archivos. El [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) no comprueba estos archivos; la versión v26.10.02 del 2 de octubre de 2026 no incluye una comprobación correspondiente (a 9 de octubre de 2026). Por ello, la verificación se realiza manualmente.

### 1. Comprobar el servidor

El siguiente script no modifica nada en el sistema. Lee la ruta de instalación de la variable de entorno `ExchangeInstallPath` y muestra los valores que KB5130098 indica como requisitos.

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
<summary>Opciones explicadas</summary>

| Línea | Efecto |
|---|---|
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Construye la ruta al directorio donde deben ubicarse los archivos de reglas. |
| `Get-Item ...\ExSetup.exe` | Lee el archivo cuya versión indica la compilación de Exchange instalada. |
| `VersionInfo.FileVersion` | Versión del archivo obtenida de los recursos de la EXE o DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcula el hash SHA256 de la `korwbrkr.dll` instalada. |
| `Test-Path ...` | Devuelve `True` si el archivo de reglas respectivo ya existe. |

</details>

La solución alternativa solo está permitida si todos los valores coinciden:

| Valor | Esperado |
| --- | --- |
| `ExSetup` | 15.02.2562.049 o 15.02.2562.053 |
| `KorwbrkrVersion` | 16.0.5194.1000 |
| `KorwbrkrBytes` | 326544 |
| `KorwbrkrSha256` | 1C6BD8E144BA677EBCC83323AE59DB3881918170F9B3A5189B44611558B92C61 |
| `KoTokenRule`, `KoComplexRule` | ambos `False` |

Si la compilación, la versión o el hash de la DLL difieren, Microsoft remite al soporte. Si ya existe uno de los dos archivos de reglas, deténgase y no sobrescriba nada.

### 2. Obtener los archivos de reglas

Los archivos proceden de `SQLEXPR_x64_ENU.exe`, el paquete x64 en inglés de SQL Server 2025 Express RTM (versión 17.0.1000.7, 748772024 bytes). Microsoft recomienda realizar este paso en una estación de trabajo de administración, no en el servidor Exchange. SQL Server no se instala: el paquete solo se descomprime y la MSI de Full-Text se extrae administrativamente. Primero compruebe la firma y el hash de la descarga:

```powershell
Get-AuthenticodeSignature .\SQLEXPR_x64_ENU.exe |
    Select-Object Status, SignerCertificate
Get-FileHash .\SQLEXPR_x64_ENU.exe -Algorithm SHA256
```

Se espera el estado `Valid` con un certificado de Microsoft y el valor SHA256 `74AA90C11202A5524E769B9BC22531BAEF22D91E9B2D2E8C3CB99E89A65C5297`. Después, descomprima y extraiga:

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
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/q` | Descomprime sin diálogos. |
| `/x:<Pfad>` | Modo de extracción del paquete SQL: descomprime los medios de instalación en la carpeta sin iniciar SQL Setup. |
| `-Wait -PassThru` | Espera a que termine el proceso y devuelve el objeto del proceso para poder leer el código de salida. |
| `msiexec /a <MSI>` | Instalación administrativa: deja el contenido de la MSI como archivos, sin instalar ni registrar nada. |
| `TARGETDIR=<Pfad>` | Carpeta de destino para los archivos extraídos. |
| `/qn` | Sin interfaz de usuario. |
| `/L*V <Datei>` | Registro detallado que muestra la causa en caso de error. |

</details>

Ambos códigos de salida deben ser `0`. Si falla la extracción, conserve `extract.log` y deténgase. A continuación, compruebe el tamaño y el hash de ambos archivos:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Archivo | Bytes | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Los archivos se encuentran en el árbol extraído bajo `...\MSSQL\Binn\ftcomponents\wordbreakers`. Transfiera únicamente estos dos archivos a una carpeta de preparación en el servidor Exchange; no transfiera medios SQL y, expresamente, tampoco la `korwbrkr.dll` del paquete SQL.

### 3. Copiar los archivos

```powershell
$native  = Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'
$staging = 'C:\Temp\KoStaging'
foreach ($f in 'ko.token.rule.bin', 'ko.complex.rule.bin') {
    Copy-Item -LiteralPath (Join-Path $staging $f) -Destination $native
    Get-FileHash -LiteralPath (Join-Path $native $f) -Algorithm SHA256
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Copy-Item -LiteralPath <Quelle>` | Copia el archivo desde la carpeta de preparación; la ruta se evalúa sin comodines. |
| `-Destination $native` | Directorio de destino junto a la `korwbrkr.dll` instalada. |
| `Get-FileHash ... -Algorithm SHA256` | Vuelve a comprobar el hash en la ubicación de destino con la tabla anterior. |

</details>

Use `Copy-Item`, no `Move-Item`. Un archivo copiado recibe los permisos heredados del directorio de destino; uno movido conserva los permisos de su carpeta de origen. Los archivos requieren los permisos de lectura normales de `Native`. No reemplace ningún otro archivo de este directorio.

### 4. Reiniciar Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

El reinicio interrumpe temporalmente la búsqueda y el procesamiento de contenido en el servidor; Microsoft prevé para ello una ventana de mantenimiento. No es necesario reiniciar el servidor. No finalice procesos de Exchange por la fuerza: si el servicio no se inicia correctamente, según Microsoft el siguiente paso es el soporte.

### 5. Comprobación

El Host Controller inicia por sí mismo el proceso `NodeRunner.exe` de ContentEngine; no lo inicie manualmente. Esta consulta muestra si se está ejecutando:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Lee todos los procesos NodeRunner con su línea de comandos. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtra el proceso de ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Muestra el ID de proceso y la hora de inicio; la hora de inicio debe ser posterior al reinicio del servicio. |

</details>

Si no aparece ningún proceso o termina repetidamente, no modifique más servidores. De lo contrario, compruebe en Outlook en la web, con un buzón cuya copia de base de datos activa esté en este servidor, si se entregan y encuentran correos nuevos, incluidos correos con texto coreano. Compruebe asimismo el servicio que estaba afectado originalmente (entrega, conexión de Outlook). Supervise por separado el retraso del índice de búsqueda: que se encuentren correos nuevos no demuestra que los elementos antiguos ya se hayan procesado. Modifique otros servidores uno por uno y solo tras comprobar correctamente el anterior.

## Instalación y tareas posteriores

Microsoft recomienda el procedimiento conocido: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la versión está desactualizada, instalar la SU, reiniciar el servidor y comprobar que todos los servicios de Exchange estén en ejecución. El Health Checker también muestra después si la SU se instaló correctamente. La Security Update Guide indica que las actualizaciones requieren reinicio.

Tras la instalación hay tres tareas posteriores:

1. Elimine el SettingOverride de wrappers `DisableBlockSharedAndUserMailboxHeaders` si está configurado (véase arriba).

2. En Exchange SE, compruebe con el script de verificación de la [sección WordBreaker](#workaround-für-den-wordbreaker-deadlock) si faltan los dos archivos de reglas y verifique si se produce el problema de libre/ocupado. Aplique las soluciones alternativas solo si se cumplen las condiciones correspondientes.

3. Para calendarios publicados con suscriptores externos, configure la regla de reescritura de URL de KB5126672 si aún no lo hizo desde la SU de agosto y las SU de octubre todavía no están instaladas.

También se mantiene desde julio la comprobación de si la mitigación de CVE-2026-42897 (M2.1.0) sigue activa; cómo eliminarla se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

Desde el 13 de octubre de 2026, según los artículos de KB, las SU de octubre sustituyen a las SU de septiembre (Exchange SE: KB5128895, compilación 15.2.2562.56). Dado que las SU son acumulativas, instale directamente la SU de octubre en todos los servidores Exchange y máquinas con Management Tools; los detalles están en el [artículo sobre la SU de octubre](/blog/exchange-security-updates-oktober-2026). Cuando esto aún no sea posible, un servidor con la SU del 8 de septiembre necesita como mínimo v2 para CVE-2026-96940. La vulnerabilidad de suplantación con CVSS 9.3 y la toma de control de buzones mediante una cuenta de usuario con pocos privilegios (CVE-2026-69380) son motivo suficiente para no esperar al próximo Patch Tuesday. Después, elimine el override de wrappers, compruebe los tres problemas conocidos y ejecute el Health Checker. El Health Checker no detecta los archivos WordBreaker ausentes; para ello es necesario el script de comprobación de la sección WordBreaker. Para Exchange 2016 y 2019, el programa ESU finaliza en octubre de 2026; ya no se puede posponer la migración a Exchange SE.

## Fuentes

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Anuncio oficial de la versión con versiones compatibles, aviso de ESU, problemas conocidos, problemas corregidos y procedimiento de instalación (consultado a través del feed RSS del Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de v2 del 2 de octubre de 2026; la única diferencia es CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Lista de CVE, problemas corregidos y los tres problemas conocidos para Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Artículo de KB para Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Artículo de KB para Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Artículo de KB para Exchange 2016 CU23, sin CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, evaluación de explotación y FAQ para las nueve CVE de septiembre y CVE-2026-96940, incluidas las compilaciones afectadas por CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Artículo de KB sobre v2 para Exchange SE; los artículos v2 para 2019 y 2016 son KB5129956 a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de v2 del 2 de octubre de 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Corrección en la SU de septiembre e instrucciones para eliminar el SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa, solución alternativa de reescritura de URL y corrección con las SU de octubre del 13 de octubre de 2026.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas, solución alternativa con SettingOverride para Exchange SE, corrección en el servicio y comandos para revertirla.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, criterios de comprobación, tamaños y hashes de archivos esperados, obtención desde SQL Server 2025 Express, reinicio del servicio, comprobación y corrección con KB5128895.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): El error de zona horaria en Exchange SE corregido con la SU de septiembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Desglose en alemán de las nueve CVE con valores CVSS y compilaciones.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Resumen en alemán con indicaciones sobre la posterior actualización de sustitución.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Código fuente del Exchange Health Checker; versión v26.10.02 del 2 de octubre de 2026 sin comprobación de los archivos WordBreaker (a 9 de octubre de 2026).

18. [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): SU de octubre que sustituye a la SU de septiembre y corrige el error .ics y el interbloqueo de WordBreaker.
