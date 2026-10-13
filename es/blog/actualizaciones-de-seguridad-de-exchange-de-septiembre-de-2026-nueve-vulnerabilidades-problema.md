---
title: "Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de los wrappers y publicada la v2"
navTitle: "Exchange SU 09/2026"
description: "La SU de septiembre corrige nueve vulnerabilidades en Exchange SE y 2019 (ocho en Exchange 2016), incluida una vulnerabilidad de suplantación con CVSS 9.3, y resuelve el problema de los wrappers en entornos híbridos. El 2 de octubre llegó una v2 con una CVE adicional; además, hay tres problemas conocidos con soluciones alternativas y un SettingOverride que ahora debe eliminarse."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Híbrido"
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
translationSourceHash: 4b2c59794f87e999c5b89865e0005faa7d5cbb36478f6b8475d5e3f121186a7d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:00:53.318Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema
---

# Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, corregido el problema de los wrappers y publicada la v2

Microsoft publicó el 8 de septiembre de 2026 actualizaciones de seguridad (SU) para Exchange Server. Corrigen nueve vulnerabilidades en Exchange SE y Exchange 2019, y ocho en Exchange 2016. Ninguna era conocida públicamente de antemano, ninguna se está explotando activamente según la Security Update Guide, y Microsoft las clasifica todas como *Important* con «Exploitation Less Likely». Sin embargo, el valor CVSS más alto, 9.3, supera claramente al del mes anterior. Este mes es especial por tres motivos: la SU corrige el problema de los *mensajes wrapper* en buzones compartidos que permanecía abierto desde junio, introduce tres problemas conocidos nuevos o persistentes y, el 2 de octubre, Microsoft publicó una **versión 2** que corrige una vulnerabilidad adicional.

## Para qué versiones de Exchange está disponible la actualización

Las SU del 8 de septiembre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, compilación 15.2.2562.49; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5121609, compilación 15.2.1748.51; solo a través del **programa ESU del período 2**.
- **Exchange Server 2019 CU14**: KB5121610, compilación 15.2.1544.46; solo a través de ESU del período 2.
- **Exchange Server 2016 CU23**: KB5121611, compilación 15.1.2507.73; solo a través de ESU del período 2.

Exchange 2016 y 2019 están fuera de soporte. Según Microsoft, las SU de mayo a octubre de 2026 solo las reciben las organizaciones inscritas en el programa ESU del período 2. Según los artículos de KB, esta autorización se extiende hasta octubre de 2026. También existe presión desde Exchange Online: desde la segunda semana de septiembre, Exchange Online limita y bloquea el flujo de correo híbrido de servidores con una versión anterior a octubre de 2025; los detalles están en el [artículo sobre la aplicación del transporte](/blog/exchange-online-transport-enforcement-hybrid-server). Según el anuncio, Exchange Online ya está protegido; aun así, en entornos híbridos todos los servidores Exchange necesitan la SU, al igual que las máquinas con las herramientas de administración de Exchange.

Puede comparar su propia versión con la lista de [números de compilación de Exchange](/tools/exchange-builds).

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

CVE-2026-55007 no afecta a Exchange 2016; la Security Update Guide solo enumera Exchange SE y 2019 CU14/CU15 para esta CVE. Un detalle sobre la documentación: en los artículos de KB de las cuatro SU de septiembre falta CVE-2026-69380 en la lista de CVE. Sin embargo, la Security Update Guide indica precisamente las cuatro compilaciones de septiembre como corrección para esta CVE (a fecha de 7 de octubre de 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** tiene el valor más alto, CVSS 9.3. Según Microsoft, un atacante no autenticado puede enviar una invitación de calendario manipulada con un enlace de reunión malicioso; si la persona destinataria abre la reunión y selecciona el enlace para unirse, se activa el Cross-Site Scripting. Por tanto, se necesita interacción del usuario, pero no una cuenta en la organización.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (elevación de privilegios, CVSS 8.1) solo requiere una cuenta con pocos permisos y un buzón asignado. Según las FAQ de la Security Update Guide, un atacante puede aprovechar debilidades en la validación de solicitudes y tokens de identidad para hacerse pasar por otro usuario y tomar control de los buzones de todos los usuarios de Exchange: leer y enviar correos, y descargar adjuntos. Basta una única cuenta de usuario comprometida como punto de partida.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (elevación de privilegios, CVSS 9.1) conduce al mismo resultado, la toma de control de todos los buzones, pero requiere pertenecer a un grupo de roles con privilegios elevados.

La situación de partida es diferente para las dos vulnerabilidades de ejecución remota de código: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requiere una cuenta autenticada con pocos permisos, mientras que [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) puede activarse sin iniciar sesión mediante un adjunto de Visio manipulado, aunque según Microsoft exige una memoria disponible persistentemente escasa en el sistema de destino. Las otras cuatro vulnerabilidades son: CVE-2026-69378 (DoS mediante recursión no controlada, sin inicio de sesión), CVE-2026-69361 (SSRF; el servidor envía solicitudes HTTP a sistemas internos o de loopback), CVE-2026-69375 (un atacante autenticado puede reemplazar el contenido de archivos) y CVE-2026-69382 (divulgación de credenciales mediante un algoritmo criptográfico débil; requiere una cookie de autenticación ya obtenida).

## Versión 2 del 2 de octubre: incorporada CVE-2026-96940

El 2 de octubre de 2026, Microsoft publicó la «versión 2» de las SU de septiembre. Según el anuncio, la única diferencia respecto a la primera versión es la corrección adicional de **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Esta vulnerabilidad de elevación de privilegios tiene CVSS 8.8, no es conocida públicamente ni se ha explotado, pero Microsoft es la única de las diez CVE que la evalúa con **«Exploitation More Likely»**. Un atacante autenticado puede usarla para acceder a buzones ajenos de la misma organización y leer correos con sus adjuntos. Exchange Online ya está corregido del lado del servidor.

| Versión | KB | Compilación v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En la práctica, esto significa que la corrección de CVE-2026-96940 solo está incluida en las compilaciones v2. Los servidores en los que ya se ejecuta la SU del 8 de septiembre también necesitan v2. Quien aplique los parches ahora puede instalar directamente v2, ya que las SU son acumulativas. Los artículos de KB no indican expresamente si los servidores con la primera SU de septiembre deben instalar v2 obligatoriamente; dado que la nueva CVE solo se corrige con v2, esa es la interpretación lógica.

## Problema de los wrappers corregido: elimine ahora el SettingOverride

El problema conocido desde la SU de junio, por el que aparecen *mensajes wrapper* en la bandeja de entrada de buzones compartidos en entornos híbridos, se ha corregido con la SU de septiembre en las cuatro versiones. En el [artículo de agosto](/blog/exchange-security-updates-august-2026) todavía se indicaba que el SettingOverride documentado como solución alternativa podía mantenerse. Tras instalar la SU de septiembre ocurre lo contrario: Microsoft recomienda, en el artículo de soporte correspondiente, comprobar y eliminar el override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Comando | Efecto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Comprueba si el override de solución alternativa está configurado en la organización. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Elimina el override una vez instalada la SU de septiembre. |

</details>

Si el primer comando indica que no se encontró el objeto `DisableBlockSharedAndUserMailboxHeaders`, según Microsoft no hay que hacer nada más.

Para Exchange SE, la SU también corrige un error en las consultas híbridas de disponibilidad mediante Microsoft Graph: los usuarios locales veían las horas ocupadas de los buzones de Exchange Online desplazadas según su propio desfase UTC, sin mensaje de error.

## Problemas conocidos

**Los calendarios publicados (.ics) devuelven HTTP 500 a las aplicaciones de calendario.** El problema existe desde la SU de agosto (Exchange SE a partir de la compilación 15.2.2562.46, así como Exchange 2019 y 2016) y tampoco se ha corregido en la SU de septiembre ni en v2. Las suscripciones a calendarios publicados de forma anónima ya no se actualizan; la misma URL funciona en el navegador. Según Microsoft, la causa es que Exchange identifica a los clientes por el User-Agent; las aplicaciones de calendario entran sin identificador de navegador en una ruta de código que la SU de agosto desactivó. Como solución alternativa, Microsoft describe una regla de reescritura de URL en el sitio «Exchange Back End» de IIS, que añade el parámetro `layout=premium` a las solicitudes .ics bajo `/owa/calendar/`; para ello debe estar instalado el módulo IIS URL Rewrite. Los pasos exactos, mediante el administrador de IIS o directamente en `applicationHost.config`, están en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft no indica una fecha para la corrección (a fecha de 7 de octubre de 2026).

**Disponibilidad para buzones delegados en entornos híbridos (solo Exchange SE).** Si la consulta de disponibilidad está configurada exclusivamente mediante la API de Graph, las consultas para buzones de Exchange Online mediante acceso local delegado fallan. Outlook informa «Your server location could not be determined», OWA muestra «No information» y en los registros de EWS aparece `(403) Forbidden`. La solución alternativa documentada redirige de nuevo las consultas mediante EWS en lugar de Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | El override que controla el reenvío de las consultas de disponibilidad mediante Microsoft Graph. |
| `-Parameters "Enabled=False"` | Desactiva la ruta de Graph; las consultas vuelven a realizarse mediante EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige la llamada de diagnóstico al servicio de topología. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga la Variant Configuration para que el override se aplique sin esperar. |

</details>

En el anuncio de v2, Microsoft incluye este problema entre los problemas corregidos. Quienes hayan configurado la solución alternativa deberían comprobar, tras instalar v2, en el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) si debe revertirse; a fecha de 7 de octubre de 2026 aún no figuraban instrucciones al respecto.

**Interbloqueo de ContentEngine por archivos de reglas WordBreaker coreanos ausentes (solo Exchange SE).** Con la SU de septiembre (compilación 15.2.2562.49 y compilación v2 15.2.2562.53), los archivos de reglas del WordBreaker coreano actualizado no se instalan. Las consecuencias son resultados de búsqueda ausentes, entrega de correo retrasada y clientes de Outlook o MAPI que se bloquean o pierden la conexión. [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) indica que se ven afectados los entornos que procesan correos con contenido en coreano. Esto no depende del idioma de los propios usuarios: los mensajes con texto coreano también llegan a organizaciones sin correspondencia en coreano, por ejemplo como spam o boletines. Por ello, los entornos suizos no quedan excluidos de entrada. Microsoft sigue investigando el problema; la comprobación y la solución alternativa se detallan en la siguiente sección.

## Solución alternativa para el interbloqueo de WordBreaker

La solución alternativa de KB5130098 añade dos archivos de reglas del paquete de SQL Server 2025 Express RTM. Microsoft la vincula a condiciones: no debe aplicarse solo por los síntomas, sino únicamente en servidores que cumplan todos los criterios de comprobación y en los que falten **ambos** archivos. El [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) no comprueba estos archivos; la versión v26.10.02 del 2 de octubre de 2026 no contiene la comprobación correspondiente (a fecha de 9 de octubre de 2026). Por ello, la comprobación se realiza manualmente.

### 1. Comprobar el servidor

El siguiente script no modifica nada en el sistema. Lee la ruta de instalación desde la variable de entorno `ExchangeInstallPath` y muestra los valores que KB5130098 indica como requisito.

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
| `Join-Path $env:ExchangeInstallPath 'Bin\Search\Ceres\Native'` | Crea la ruta al directorio donde deben estar los archivos de reglas. |
| `Get-Item ...\ExSetup.exe` | Lee el archivo cuya versión indica la compilación de Exchange instalada. |
| `VersionInfo.FileVersion` | Versión del archivo procedente de los recursos de la EXE o DLL. |
| `Get-FileHash -LiteralPath ... -Algorithm SHA256` | Calcula el hash SHA256 de `korwbrkr.dll` instalada. |
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

Si la compilación, la versión o el hash de la DLL difieren, Microsoft remite al soporte. Si uno de los dos archivos de reglas ya está presente, deténgase y no sobrescriba nada.

### 2. Obtener los archivos de reglas

Los archivos proceden de `SQLEXPR_x64_ENU.exe`, el paquete x64 en inglés de SQL Server 2025 Express RTM (versión 17.0.1000.7, 748772024 bytes). Microsoft recomienda realizar este paso en una estación de trabajo de administración, no en el servidor Exchange. SQL Server no se instala: el paquete solo se descomprime y el MSI de Full-Text solo se extrae administrativamente. Compruebe primero la firma y el hash de la descarga:

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
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `/q` | Descomprime sin diálogos. |
| `/x:<Pfad>` | Modo de extracción del paquete SQL: descomprime los medios de instalación en la carpeta sin iniciar SQL Setup. |
| `-Wait -PassThru` | Espera a que finalice el proceso y devuelve el objeto de proceso para poder leer el código de salida. |
| `msiexec /a <MSI>` | Instalación administrativa: deposita el contenido del MSI como archivos, sin instalar ni registrar nada. |
| `TARGETDIR=<Pfad>` | Directorio de destino para los archivos extraídos. |
| `/qn` | Sin interfaz de usuario. |
| `/L*V <Datei>` | Registro detallado que muestra la causa en caso de error. |

</details>

Ambos códigos de salida deben ser `0`. Si la extracción falla, guarde `extract.log` y deténgase. A continuación, compruebe el tamaño y el hash de ambos archivos:

```powershell
Get-ChildItem "$work\Files" -Recurse -Include 'ko.token.rule.bin', 'ko.complex.rule.bin' |
    Select-Object FullName, Length,
        @{ n = 'SHA256'; e = { (Get-FileHash -LiteralPath $_.FullName -Algorithm SHA256).Hash } }
```

| Archivo | Bytes | SHA256 |
| --- | --- | --- |
| `ko.token.rule.bin` | 56132 | 8F2BD853593913EB8F73DCD4FCAC4216F216A0FF76A4569DF071BE3C36773010 |
| `ko.complex.rule.bin` | 717792 | 0390D1E9A76EF33283025CF8F164430E311584B9535949C4EA1A74B6BB107B87 |

Los archivos se encuentran en el árbol extraído bajo `...\MSSQL\Binn\ftcomponents\wordbreakers`. Transfiera únicamente estos dos archivos a una carpeta de staging en el servidor Exchange; no transfiera medios SQL ni, expresamente, `korwbrkr.dll` del paquete SQL.

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
| `Copy-Item -LiteralPath <Quelle>` | Copia el archivo desde la carpeta de staging; la ruta se evalúa sin caracteres comodín. |
| `-Destination $native` | Directorio de destino junto a `korwbrkr.dll` instalada. |
| `Get-FileHash ... -Algorithm SHA256` | Vuelve a comprobar el hash en el destino con la tabla anterior. |

</details>

Utilice `Copy-Item`, no `Move-Item`. Un archivo copiado recibe los permisos heredados del directorio de destino; uno movido conserva los permisos de su carpeta de origen. Los archivos necesitan los permisos de lectura normales de `Native`. No reemplace otros archivos de este directorio.

### 4. Reiniciar Search Host Controller

```powershell
Restart-Service -Name HostControllerService
```

El reinicio interrumpe temporalmente la búsqueda y el procesamiento de contenido en el servidor; Microsoft prevé una ventana de mantenimiento para ello. No es necesario reiniciar el propio servidor. No finalice procesos de Exchange de forma forzada: si el servicio no se inicia correctamente, según Microsoft el siguiente paso es contactar con soporte.

### 5. Comprobación

Host Controller inicia por sí mismo el proceso `NodeRunner.exe` de ContentEngine; no lo inicie manualmente. Esta consulta muestra si está en ejecución:

```powershell
Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'" |
    Where-Object CommandLine -like '*ContentEngineNode1*' |
    Select-Object ProcessId, CreationDate
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-CimInstance Win32_Process -Filter "Name = 'NodeRunner.exe'"` | Lee todos los procesos NodeRunner junto con su línea de comandos. |
| `Where-Object CommandLine -like '*ContentEngineNode1*'` | Filtra por el proceso de ContentEngine. |
| `Select-Object ProcessId, CreationDate` | Muestra el ID del proceso y la hora de inicio; la hora de inicio debe ser posterior al reinicio del servicio. |

</details>

Si no aparece ningún proceso o este finaliza repetidamente, no trabaje en más servidores. De lo contrario, compruebe en Outlook en la Web, con un buzón cuya copia activa de base de datos esté en este servidor, si se entregan y encuentran nuevos correos, incluidos los correos con texto coreano. Compruebe también el servicio que estaba afectado originalmente (entrega, conexión de Outlook). Observe por separado el retraso del índice de búsqueda: que se encuentren los correos nuevos no demuestra que los elementos antiguos ya se hayan procesado. Procese otros servidores uno por uno y solo después de comprobar correctamente el anterior.

## Instalación y tareas posteriores

Microsoft recomienda el procedimiento habitual: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la versión está desactualizada, instalar la SU, reiniciar el servidor y comprobar que todos los servicios de Exchange estén en ejecución. Después, Health Checker también muestra si la SU se instaló correctamente. La Security Update Guide indica que las actualizaciones requieren reinicio.

Tras la instalación hay tres tareas pendientes:

1. Elimine el SettingOverride de wrapper `DisableBlockSharedAndUserMailboxHeaders` si está configurado (véase arriba).

2. En Exchange SE, compruebe con el script de comprobación de la [sección WordBreaker](#workaround-für-den-wordbreaker-deadlock) si faltan los dos archivos de reglas y compruebe si se produce el problema de disponibilidad. Aplique las soluciones alternativas solo cuando se cumplan las condiciones correspondientes.

3. Para calendarios publicados con suscriptores externos, configure la regla de reescritura de URL de KB5126672 si todavía no se hizo desde la SU de agosto.

Además, desde julio permanece la comprobación de si la mitigación CVE-2026-42897 (M2.1.0) sigue activa; cómo eliminarla se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

Instale directamente las compilaciones v2 del 2 de octubre en todos los servidores Exchange y las máquinas con herramientas de administración; los servidores con la SU del 8 de septiembre necesitan además v2 para CVE-2026-96940. La vulnerabilidad de suplantación con CVSS 9.3 y la toma de control de buzones mediante una cuenta de usuario con pocos permisos (CVE-2026-69380) son razones suficientes para no esperar al próximo Patch Tuesday. Después, elimine el override de wrapper, compruebe los tres problemas conocidos y ejecute Health Checker. Health Checker no detecta los archivos WordBreaker ausentes; para ello se necesita el script de comprobación de la sección WordBreaker. El programa ESU para Exchange 2016 y 2019 termina en octubre de 2026; ya no puede aplazarse la migración a Exchange SE.

## Fuentes

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Anuncio oficial de la versión con las versiones compatibles, aviso de ESU, problemas conocidos, problemas corregidos y proceso de instalación (consultado a través del feed RSS de Exchange Team Blogs).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de la v2 del 2 de octubre de 2026; la única diferencia es CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Lista de CVE, problemas corregidos y los tres problemas conocidos de Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Artículo de KB para Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Artículo de KB para Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Artículo de KB para Exchange 2016 CU23, sin CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, evaluación de explotación y FAQ sobre las nueve CVE de septiembre y CVE-2026-96940, incluidas las compilaciones afectadas para cada CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Artículo de KB de la v2 para Exchange SE; los artículos v2 para 2019 y 2016 son KB5129956 a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de la v2 del 2 de octubre de 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Corrección en la SU de septiembre e instrucciones para eliminar el SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa y solución alternativa de reescritura de URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas y solución alternativa con SettingOverride para Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, criterios de comprobación, tamaños y hashes esperados de archivos, obtención desde SQL Server 2025 Express, reinicio del servicio y comprobación.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Error de zona horaria de Exchange SE corregido con la SU de septiembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Desglose en alemán de las nueve CVE con valores CVSS y compilaciones.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Resumen en alemán con referencias a la posterior actualización de reemplazo.

17. [microsoft/CSS-Exchange – GitHub](https://github.com/microsoft/CSS-Exchange): Código fuente de Exchange Health Checker; versión v26.10.02 del 2 de octubre de 2026 sin comprobación de los archivos WordBreaker (a fecha de 9 de octubre de 2026).
