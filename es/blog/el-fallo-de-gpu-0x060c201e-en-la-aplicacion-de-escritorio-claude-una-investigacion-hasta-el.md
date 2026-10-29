---
title: "Claude Desktop se bloquea constantemente: «GPU process gone» con código de salida 101457950, causa y solución"
navTitle: "Bloqueo de Claude Desktop"
description: "La aplicación Claude Desktop en Windows se cierra por completo con «GPU process gone: exitCode 101457950» (0x060C201E), a menudo seguido del diálogo de reparación de la aplicación Store. La cadena completa de causas: Code Integrity bloquea vk_swiftshader.dll, la cadena de respaldo de Chromium se agota y el cierre automático integrado finaliza la aplicación. Con una solución permanente (cambiar a la instalación clásica sin MSIX), autodiagnóstico mediante el registro de eventos y análisis hasta el minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min de lectura"
themen:
  - claude
slug: "el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:30:36.775Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el
---

La aplicación Claude Desktop en Windows se cierra sin mensaje de error, todas las sesiones de Claude Code en curso desaparecen y, a veces, la aplicación solo vuelve a iniciarse tras «Reparar» desde la configuración de Windows. En el registro de la aplicación aparece entonces esta línea:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 en hexadecimal es `0x060C201E`. Si encuentra esta firma en su registro, está en el lugar adecuado: este artículo documenta la cadena completa de causas de este bloqueo, las medidas inmediatas con las que la aplicación vuelve a funcionar de forma estable y el autodiagnóstico con el que puede confirmar el hallazgo en su propio sistema en dos minutos. Se ven afectadas las instalaciones MSIX (desde Microsoft Store o mediante configuración MSIX) con todos los fabricantes de GPU, desde las GPU integradas de Intel hasta NVIDIA y AMD; el hardware, adelantando la conclusión, no es la causa. La instalación clásica sin MSIX no se ve afectada, y esa es precisamente la solución.

## La solución en breve: cambiar a la instalación clásica

El error real se encuentra en el paquete de instalación MSIX y solo Anthropic puede corregirlo (la incidencia [#81341](https://github.com/anthropics/claude-code/issues/81341) sigue abierta a fecha de 25.10.2026; a principios de septiembre de 2026, los informes en la incidencia confirmaron el error también en la versión 1.40609.1). Sin embargo, la misma aplicación también existe como instalación clásica sin MSIX, y esta no está sujeta a la comprobación de firma AppX que finaliza el proceso de GPU. Por tanto, el cambio es la única medida que elimina completamente el bloqueo; está confirmado tanto en la incidencia [#81341](https://github.com/anthropics/claude-code/issues/81341) como en el sistema analizado aquí. Las funciones son idénticas y el canal de actualizaciones proporciona las mismas versiones para ambas variantes.

**Paso 1: descargar y ejecutar el instalador clásico.** La descarga en [claude.com/download](https://claude.com/download) proporciona un instalador Squirrel que instala la aplicación en `%LOCALAPPDATA%\AnthropicClaude` (no se requieren derechos de administrador). Desde la línea de comandos:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-L` | sigue las redirecciones HTTP hasta el archivo real |
| `-o <pfad>` | archivo de destino; aquí, la carpeta Descargas |
| `<url>` | fuente oficial del instalador; idéntica al destino de la redirección de descarga desde claude.ai |

</details>

Tras la descarga, compruebe la firma (`Get-AuthenticodeSignature`, se espera: `Valid`, emisor «Anthropic, PBC») e inicie el archivo. El instalador coloca primero una versión base anterior; el mecanismo de actualización la lleva a la versión actual, ya sea automáticamente en el primer inicio o de inmediato mediante:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `--update <url>` | descarga la versión más reciente del canal de versiones indicado y la instala como un nuevo directorio `app-<version>` |

</details>

**Paso 2: trasladar la configuración.** La versión MSIX guarda el inicio de sesión, la configuración de servidores MCP y los ajustes en su contenedor virtualizado; la aplicación clásica lee `%APPDATA%\Claude`. Copie una vez (cierre antes la aplicación MSIX; de todos modos, ambas variantes no se ejecutan simultáneamente debido a un bloqueo compartido de instancia única):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `<quelle>` | carpeta de configuración en el AppData virtualizado del paquete MSIX |
| `<ziel>` | carpeta de configuración de la instalación clásica |
| `/E` | copia todos los subdirectorios, incluidos los vacíos |
| `/XD <namen>` | omite los directorios indicados; aquí, cachés y datos de tiempo de ejecución que la nueva aplicación crea por sí misma |

</details>

Los historiales de chat no se pierden: están en la cuenta de claude.ai o (para sesiones de Claude Code) en `%USERPROFILE%\.claude` y no dependen de la instalación de la aplicación.

**Paso 3: eliminar el paquete MSIX.** De lo contrario, los accesos directos antiguos seguirán iniciando la variante que se bloquea:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Claude` | argumento posicional Name de `Get-AppxPackage`: filtra los paquetes AppX/MSIX instalados por el nombre del paquete (se permiten comodines) |
| `Remove-AppxPackage` | elimina para la cuenta de usuario actual el paquete transferido mediante la canalización |

</details>

La entrada del menú Inicio «Anthropic → Claude» pertenecerá después a la instalación clásica; habrá que volver a fijar cualquier acceso anclado a la barra de tareas.

## Si debe permanecer con el paquete MSIX

Sin cambiar, solo quedan medidas que reducen la frecuencia de los bloqueos sin eliminar la causa:

**Use con moderación el navegador integrado.** Las páginas del área de navegador/vista previa de la aplicación desencadenan el bloqueo. Quien cierre el área después de usarla en vez de dejar pestañas abiertas reduce notablemente la frecuencia de bloqueos; esta relación está documentada varias veces con cifras en el hilo de la comunidad.

**Desactive WebGPU.** Iniciar con `--disable-features=WebGPU` evita el desencadenante más frecuente. En un paquete MSIX, la ruta de instalación cambia con cada actualización; por ello, use un lanzador que la resuelva de nuevo en cada inicio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `for /f "delims="` | procesa la salida del comando línea por línea; `delims=` vacío asigna la línea completa (incluidos espacios en la ruta) a `%%i` |
| `-NoProfile` | inicia PowerShell sin scripts de perfil, para un inicio rápido y reproducible |
| `-Command` | ejecuta la expresión indicada; `(Get-AppxPackage Claude).InstallLocation` proporciona la ruta de instalación actual del paquete |
| `start ""` | inicia el programa desvinculado de la ventana por lotes; las comillas vacías son el título de ventana (aquí vacío) |
| `--disable-features=WebGPU` | modificador de Chromium: desactiva la función indicada, aquí la API WebGPU |

</details>

Esto solo funciona si la aplicación también se inicia mediante este lanzador.

La primera versión de este artículo recomendaba en primer lugar activar la aceleración por hardware mediante `isHardwareAccelerationDisabled: false` en `config.json`. Esta recomendación está desactualizada: en las versiones actuales (1.37937.x), la marca ya no existe, la aplicación se inicia de forma predeterminada con aceleración por hardware activa y, aun así, se bloquea (detalles en la actualización más abajo).

Por cierto, «Reparar» o reinstalar el paquete MSIX no soluciona el problema; solo trata el síntoma secundario (más sobre ello abajo). Actualizar los controladores gráficos también es un esfuerzo inútil.

## Autodiagnóstico: confirmar el hallazgo en su propio sistema

Bastan dos comprobaciones. La primera, la firma del bloqueo en el registro de la aplicación:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Path` | archivo que se debe buscar; aquí, el registro principal de la aplicación |
| `-Pattern` | patrón de búsqueda (expresión regular); muestra todas las líneas con la firma del bloqueo |

</details>

La segunda, y esta es la prueba real, el registro CodeIntegrity de Windows:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-FilterHashtable` | filtra ya durante la consulta: `LogName` designa el registro de eventos, `Id` el ID de evento 3033 (bloqueo de Code Integrity) |
| `-MaxEvents 30` | limita la consulta a los 30 resultados más recientes |
| `Where-Object { … -match 'claude' }` | conserva solo los eventos cuyo texto de mensaje contiene la ruta de la aplicación |
| `Select-Object TimeCreated, Message` | reduce la salida a la marca de tiempo y el mensaje para compararlos con los momentos de bloqueo |

</details>

En los sistemas afectados encontrará entradas del evento 3033 cuyas marcas de tiempo coinciden al segundo con los momentos de bloqueo, con este mensaje:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

En el sistema analizado aquí, siete de siete bloqueos durante tres semanas coincidieron al segundo con uno de estos eventos, incluido un bloqueo de control provocado deliberadamente.

## La cadena completa de causas

El bloqueo es el último eslabón de una cadena de cuatro elementos que resultan conjuntamente de dos análisis: el rastro de Code Integrity de la incidencia de la comunidad [#81698](https://github.com/anthropics/claude-code/issues/81698) y un análisis propio de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Eslabón 1: una página del navegador integrado necesita renderizado por software.** Un desencadenante típico es una llamada WebGPU (`navigator.gpu.requestAdapter()`), reconocible en el registro de la ventana por esta advertencia inmediatamente antes del bloqueo:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Si la aplicación se ejecuta sin aceleración por hardware, la ruta pasa obligatoriamente por la implementación Vulkan por software SwiftShader: el proceso de GPU intenta cargar `vk_swiftshader.dll` incluido.

**Eslabón 2: Windows Code Integrity bloquea la propia DLL de la aplicación.** El proceso de GPU se ejecuta con la política de protección «MicrosoftSignedOnly» (verificable mediante `Get-ProcessMitigation`). Para que una aplicación Store pueda cargar sus propias DLL firmadas por el fabricante, el paquete MSIX debe incluir un catálogo de firmas `AppxMetadata\CodeIntegrity.cat`. Precisamente ese archivo falta en el paquete distribuido, como han demostrado miembros de la comunidad mediante la inspección del archivo MSIX. Consecuencia: falla la comprobación de firma, Windows registra el evento 3033 y finaliza el proceso de GPU de forma forzosa. El código de salida `0x060C201E` es un error de integridad AppX del cargador de Windows, no un código de Chromium; por eso no se encuentra en ninguna fuente de Chromium y por eso el proceso de GPU tampoco deja un volcado de bloqueo: no hay ninguna excepción que se pueda volcar.

**Eslabón 3: la cadena de respaldo de Chromium se agota.** Cuando el proceso de GPU se bloquea, Chromium retrocede un nivel de renderizado: GL por hardware, luego GL por software y después compositor de pantalla puro. Solo cuando no queda ningún nivel, entra en acción el cierre automático integrado. En el código fuente de la versión incluida (Chromium 148.0.7778.280 en Electron 42.9.2) aparece literalmente así:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Eslabón 4: el proceso principal se finaliza deliberadamente.** Este `LOG(FATAL)` es el momento en que «la aplicación se bloquea». Lo demuestra un minidump del proceso principal: `EXCEPTION_BREAKPOINT` (un `int3` deliberado, no un error de controlador), ni una sola DLL de controlador gráfico en el proceso y, en memoria, en texto claro:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Que este volcado exista siquiera fue la parte más difícil del análisis: la integración de Sentry de la aplicación consume los volcados Crashpad en el siguiente inicio de la aplicación, los envía a la telemetría del fabricante y los elimina localmente. Por ello, la carpeta Crashpad siempre está vacía para el usuario. La solución es un observador independiente del árbol de procesos de la aplicación (iniciado mediante WMI para que el bloqueo de la aplicación no lo finalice también), que examina la base de datos Crashpad cada 200 milisegundos en busca de `*.dmp` y copia inmediatamente los hallazgos antes de que se eliminen. El paquete Python `minidump` realiza la evaluación, sin necesidad de WinDbg.

## Por qué «desactivar la aceleración por hardware» lo empeora todo

La cadena también explica el hallazgo más contraintuitivo. La aceleración por hardware desactivada tiene aquí dos efectos fatales a la vez. Primero, fuerza la ruta de SwiftShader, es decir, precisamente el intento de carga de DLL que Code Integrity bloquea; con la aceleración por hardware activa, `vk_swiftshader.dll` apenas se necesita. Segundo, el proceso de GPU se inicia ya en el extremo inferior de la cadena de respaldo: basta un solo bloqueo para que actúe el eslabón 4. Esto también explica la observación del hilo de la comunidad de que un bloqueo de Code Integrity a veces no tiene consecuencias y otras veces finaliza la aplicación: depende de cuántos niveles de respaldo le queden al proceso de navegador.

Especialmente desafortunado: la aplicación conocía una desactivación automática de la aceleración por hardware tras problemas (`isHardwareAccelerationAutoDisabled`). Pensada como medida de estabilidad, llevaba a los sistemas afectados precisamente a la configuración en la que el siguiente bloqueo cuesta toda la aplicación.

## Actualización 27.08.2026: la aceleración por hardware por sí sola no basta

La primera versión de este artículo recomendaba la aceleración por hardware activa como la medida inmediata más eficaz, y durante dos días la aplicación no se bloqueó con ella. Luego llegó la actualización automática a 1.37937.3 y, con ella, tres bloqueos en una tarde, cada uno con el conocido evento 3033 relativo a `vk_swiftshader.dll`. De ello se desprenden dos hallazgos:

Primero, el catálogo de firmas ausente también falta en el paquete MSIX actual; el problema fundamental permanece inalterado en 1.37937.3.

Segundo, la aceleración por hardware activa solo protege estadísticamente: alarga la cadena de respaldo, pero no evita que Chromium, bajo carga o tras un error del proceso de GPU por hardware, la recorra hasta el nivel SwiftShader. En cuanto esto ocurre, Code Integrity bloquea la DLL y la cadena puede agotarse de todos modos. Además, las marcas de configuración `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` han desaparecido de `config.json` en 1.37937.x; ya no es posible fijar allí el ajuste.

Por tanto, la única solución fiable fue el cambio a la instalación clásica descrito arriba. Desde el cambio en el sistema analizado aquí: misma versión de la aplicación, uso idéntico incluido el área de navegador, ni un solo evento 3033 ni ningún bloqueo.

## El síntoma secundario: el bucle de reparación

El fallo de Code Integrity tiene un efecto secundario que muchos afectados consideran un problema independiente: tras el incidente, Windows clasifica en parte el paquete de la aplicación como «Modified, NeedsRemediation». Entonces la aplicación deja de iniciarse hasta que se restablece mediante Configuración → Aplicaciones → Claude → Opciones avanzadas → «Reparar». Quien, por tanto, «debe reparar constantemente la aplicación» ve el mismo problema de fondo, solo un eslabón más adelante: la reparación corrige el estado del paquete, no la causa; el siguiente bloqueo llegará con el siguiente intento de carga de DLL bloqueado.

## Estado de los informes

La causa de empaquetado se ha comunicado como [#81341](https://github.com/anthropics/claude-code/issues/81341), el hilo recopilatorio con las pruebas de la comunidad es [#81698](https://github.com/anthropics/claude-code/issues/81698), el análisis de minidump con la explicación de la cadena de respaldo es [#89250](https://github.com/anthropics/claude-code/issues/89250) y otro informe detallado, incluido el bucle de reparación, es [#80444](https://github.com/anthropics/claude-code/issues/80444). La corrección real, un catálogo de firmas completo en el paquete MSIX, corresponde a Anthropic. A fecha de 25.10.2026, #81341 sigue abierta y los informes de principios de septiembre de 2026 confirman el error también en la versión 1.40609.1. #80444 se cerró el 15 de septiembre de 2026 sin indicar una versión corregida, y #89250 el 20 de octubre de 2026 automáticamente por inactividad; ninguna de las dos incidencias menciona una corrección. Hasta entonces, aplique lo siguiente: cambie a la instalación clásica; quien deba permanecer con el paquete MSIX debe cerrar disciplinadamente el área de navegador y, si es necesario, desactivar WebGPU mediante una marca. En el sistema analizado aquí, la aplicación no ha vuelto a bloquearse desde el cambio a la instalación clásica, sin un solo evento 3033 adicional.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: la incidencia #81341 sigue abierta a fecha de 25.10.2026 y el error también está confirmado en la versión 1.40609.1. Las incidencias #80444 y #89250 se cerraron sin mencionar una corrección.

## Fuentes

1.  [Incidencia de GitHub #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): El hilo recopilatorio con las pruebas de la comunidad sobre la cadena de Code Integrity, los datos de distintos fabricantes y la correlación con el panel del navegador.

2.  [Incidencia de GitHub #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La causa de empaquetado; falta el catálogo CodeIntegrity en MSIX.

3.  [Incidencia de GitHub #89250: Análisis de minidump de la finalización de la aplicación](https://github.com/anthropics/claude-code/issues/89250): La segunda mitad de la cadena descrita aquí, con el método de captura de volcados y propuestas de corrección.

4.  [Incidencia de GitHub #80444: Bloqueo de GPU con análisis forense y bucle de reparación](https://github.com/anthropics/claude-code/issues/80444): Informe individual detallado con cronologías, análisis del registro de eventos y el hallazgo de que cada bloqueo lleva el paquete al estado «Modified».

5.  [Claude Desktop: página oficial de descargas](https://claude.com/download): Fuente del instalador clásico de Windows (x64 y ARM64).

6.  [Código fuente de Chromium: gpu_data_manager_impl_private.cc (etiqueta 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La función IntentionallyCrashBrowserForUnusableGpuProcess y la lógica de respaldo.

7.  [Documentación de Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): El evento con el que una aplicación Electron puede observar bloqueos de procesos de GPU y adoptar sus propias medidas.

8.  [Paquete Python minidump](https://pypi.org/project/minidump/): Herramienta de análisis de volcados (registro de excepciones, lista de módulos, cadenas de memoria).

9.  [webgpureport.org](https://webgpureport.org/): Página de diagnóstico de WebGPU; sirvió como desencadenante mínimo para el bloqueo de control y para la prueba comparativa en Chromium actual.
