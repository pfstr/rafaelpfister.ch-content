---
title: "Claude Desktop se bloquea continuamente: «GPU process gone» con código de salida 101457950, causa y solución"
navTitle: "Bloqueo de Claude Desktop"
description: "La aplicación Claude Desktop para Windows se cierra por completo con «GPU process gone: exitCode 101457950» (0x060C201E), a menudo seguido por el diálogo de reparación de la aplicación Store. La cadena completa de causas: Code Integrity bloquea vk_swiftshader.dll, la cadena de respaldo de Chromium se agota y el cierre automático integrado finaliza la aplicación. Con solución inmediata, autodiagnóstico mediante el registro de eventos y análisis hasta el minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min de lectura"
themen:
  - claude
slug: "el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 2cf7e9455d4d9b5c148e7b55fd0433206810dc26e53bacb85e1d2dc82a0444c6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:08:01.542Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el
---

La aplicación Claude Desktop para Windows se cierra sin mensaje de error, todas las sesiones de Claude Code en curso desaparecen y, a veces, la aplicación solo vuelve a iniciarse después de «Reparar» en la configuración de Windows. En el registro de la aplicación aparece entonces esta línea:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 en hexadecimal es `0x060C201E`. Si encuentra esta firma en su registro, está en el lugar correcto: este artículo documenta la cadena completa de causas de este bloqueo, las medidas inmediatas con las que la aplicación vuelve a funcionar de forma estable y el autodiagnóstico con el que puede confirmar el hallazgo en su propio sistema en dos minutos. Se ven afectadas las instalaciones de Microsoft Store (MSIX) en todos los fabricantes de GPU, desde GPU integradas de Intel hasta NVIDIA y AMD; el hardware, adelantándonos a ello, no es la causa.

## La solución en breve

El error real se encuentra en el paquete de instalación de la aplicación y solo puede ser corregido por Anthropic (pendiente a fecha de 25.08.2026, incidencia [#81341](https://github.com/anthropics/claude-code/issues/81341)). Hasta entonces, tres medidas hacen que la aplicación sea estable, en orden de eficacia:

**1. Active la aceleración por hardware.** Compruebe estos dos valores en el archivo `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` y, si es necesario, establézcalos en `false` (cierre la aplicación antes y reiníciela después):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

Suena paradójico, porque desactivar la aceleración por hardware suele ser la opción más estable. Con este error sucede lo contrario, y la cadena de causas más abajo explica por qué: el ajuste decide si un bloqueo del proceso de GPU solo cuesta un nivel de respaldo o toda la aplicación.

**2. Use con moderación el navegador integrado.** Las páginas en el área de navegador/vista previa de la aplicación desencadenan el bloqueo. Quien cierre el área después de usarla en vez de dejar pestañas aparcadas reduce drásticamente la frecuencia de bloqueos; esta relación está documentada repetidamente con cifras en el hilo de la comunidad.

**3. Opcional: desactive WebGPU.** Iniciar con `--disable-features=WebGPU` evita por completo el desencadenante más frecuente. En una aplicación Store, la ruta de instalación cambia con cada actualización; por ello, se utiliza un lanzador que la resuelve de nuevo en cada inicio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

La desventaja: esto solo funciona si la aplicación se inicia mediante este lanzador. La medida 1 funciona en cada inicio.

Por cierto, «Reparar» o reinstalar la aplicación no soluciona el problema; solo trata el síntoma posterior (más información abajo). Actualizar los controladores gráficos también es un esfuerzo inútil.

## Autodiagnóstico: confirmar el hallazgo en el propio sistema

Bastan dos comprobaciones. Primero, la firma del bloqueo en el registro de la aplicación:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Segundo, y esta es la prueba real, el registro CodeIntegrity de Windows:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

En los sistemas afectados encontrará entradas del evento 3033, cuyas marcas de tiempo coinciden al segundo con las horas de bloqueo, con este mensaje:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

En el sistema analizado aquí, siete de siete bloqueos a lo largo de tres semanas coincidieron al segundo con uno de esos eventos, incluido un bloqueo de control provocado deliberadamente.

## La cadena completa de causas

El bloqueo es el último eslabón de una cadena de cuatro elementos que revelan conjuntamente dos análisis: el rastro de Code Integrity de la incidencia de la comunidad [#81698](https://github.com/anthropics/claude-code/issues/81698) y un análisis propio de minidump ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Eslabón 1: una página en el navegador integrado necesita renderizado por software.** El desencadenante típico es una llamada a WebGPU (`navigator.gpu.requestAdapter()`), reconocible en el registro de la ventana por esta advertencia inmediatamente antes del bloqueo:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

Si la aplicación se ejecuta sin aceleración por hardware, la ruta pasa obligatoriamente por la implementación de software Vulkan SwiftShader: el proceso de GPU intenta cargar la `vk_swiftshader.dll` incluida.

**Eslabón 2: Windows Code Integrity bloquea la propia DLL de la aplicación.** El proceso de GPU se ejecuta con la política de endurecimiento «MicrosoftSignedOnly» (verificable mediante `Get-ProcessMitigation`). Para que una aplicación Store pueda cargar sus propias DLL firmadas por el fabricante, el paquete MSIX debe incluir un catálogo de firmas `AppxMetadata\CodeIntegrity.cat`. Precisamente ese archivo falta en el paquete distribuido, como han demostrado miembros de la comunidad mediante la inspección del archivo MSIX. Como consecuencia, falla la verificación de firma, Windows registra el evento 3033 y finaliza el proceso de GPU de forma forzada. El código de salida `0x060C201E` es un error de integridad de AppX del cargador de Windows, no un código de Chromium; por eso no aparece en ninguna fuente de Chromium y por eso el proceso de GPU tampoco deja un crash dump: no hay ninguna excepción que se pueda volcar.

**Eslabón 3: la cadena de respaldo de Chromium se agota.** Si el proceso de GPU se bloquea, Chromium retrocede un nivel de renderizado: GL por hardware, después GL por software y luego el compositor de pantalla puro. Solo cuando ya no queda ningún nivel entra en acción el cierre automático integrado. En el código fuente de la versión incluida (Chromium 148.0.7778.280 en Electron 42.9.2), figura literalmente así:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Eslabón 4: el proceso principal se finaliza deliberadamente.** Este `LOG(FATAL)` es el momento en que «la aplicación se bloquea». Lo demuestra un minidump del proceso principal: `EXCEPTION_BREAKPOINT` (un `int3` deliberado, no un error de controlador), ni una sola DLL de controlador gráfico en el proceso y, en la memoria, en texto sin formato:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

Que este dump exista fue la parte más difícil del análisis: la integración Sentry de la aplicación consume los dumps de Crashpad en el siguiente inicio de la aplicación, los envía a la telemetría del fabricante y los elimina localmente. Por ello, la carpeta de Crashpad siempre está vacía para el usuario. La solución es un observador independiente del árbol de procesos de la aplicación (iniciado mediante WMI para que el bloqueo de la aplicación no lo finalice también), que busca en la base de datos de Crashpad cada 200 milisegundos `*.dmp` y copia inmediatamente los hallazgos antes de que se eliminen. El análisis lo realiza el paquete Python `minidump`, sin necesidad de WinDbg.

## Por qué «desactivar la aceleración por hardware» lo empeora todo

La cadena también explica el hallazgo más contraintuitivo. La aceleración por hardware desactivada tiene aquí dos efectos fatales simultáneos. En primer lugar, fuerza la ruta de SwiftShader, es decir, precisamente el intento de carga de DLL que Code Integrity bloquea; con la aceleración por hardware activada, `vk_swiftshader.dll` apenas se necesita. En segundo lugar, el proceso de GPU se inicia ya en el extremo inferior de la cadena de respaldo: basta un solo bloqueo y el eslabón 4 entra en acción. Esto también explica la observación del hilo de la comunidad de que un bloqueo de Code Integrity a veces no tiene consecuencias y otras veces finaliza la aplicación: depende de cuántos niveles de respaldo le queden al proceso del navegador.

Especialmente desafortunado: la aplicación cuenta con una desactivación automática de la aceleración por hardware después de problemas (`isHardwareAccelerationAutoDisabled`). Pensada como medida de estabilidad, lleva a los sistemas afectados precisamente a la configuración en la que el siguiente bloqueo cuesta toda la aplicación.

## El síntoma posterior: el bucle de reparación

El fallo de Code Integrity tiene un efecto secundario que muchas personas afectadas consideran un problema independiente: tras el incidente, Windows clasifica a veces el paquete de la aplicación como «Modified, NeedsRemediation». Entonces la aplicación ya no se inicia hasta que se restablece mediante Configuración → Aplicaciones → Claude → Opciones avanzadas → «Reparar». Por tanto, quien «tiene que reparar la aplicación continuamente» está viendo el mismo problema de fondo, solo un eslabón más adelante: la reparación corrige el estado del paquete, no la causa; el siguiente bloqueo se produce en el próximo intento de carga de DLL bloqueado.

## Estado de los informes

La causa de empaquetado se ha notificado como [#81341](https://github.com/anthropics/claude-code/issues/81341), el hilo recopilatorio con las pruebas de la comunidad es [#81698](https://github.com/anthropics/claude-code/issues/81698), y el análisis de minidump con la explicación de la cadena de respaldo es [#89250](https://github.com/anthropics/claude-code/issues/89250). La corrección real, un catálogo de firmas completo en el paquete MSIX, depende de Anthropic. Hasta entonces: aceleración por hardware activada, cerrar disciplinadamente el área del navegador y, si es necesario, desactivar WebGPU mediante una bandera. En el sistema analizado aquí, la aplicación no ha vuelto a bloquearse desde la aplicación de la medida 1.

## Fuentes

1.  [Incidencia de GitHub #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): El hilo recopilatorio con las pruebas de la comunidad sobre la cadena de Code Integrity, los datos entre fabricantes y la correlación con el panel del navegador.

2.  [Incidencia de GitHub #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): La causa de empaquetado; falta el catálogo CodeIntegrity en el MSIX.

3.  [Incidencia de GitHub #89250: análisis de minidump del cierre de la aplicación](https://github.com/anthropics/claude-code/issues/89250): La segunda mitad de la cadena descrita aquí, con el método de captura de dumps y propuestas de solución.

4.  [Código fuente de Chromium: gpu_data_manager_impl_private.cc (etiqueta 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La función IntentionallyCrashBrowserForUnusableGpuProcess y la lógica de respaldo.

5.  [Documentación de Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): El evento con el que una aplicación Electron puede observar bloqueos de procesos de GPU y adoptar sus propias contramedidas.

6.  [Paquete Python minidump](https://pypi.org/project/minidump/): Herramienta para el análisis de dumps (registro de excepciones, lista de módulos, cadenas de memoria).

7.  [webgpureport.org](https://webgpureport.org/): Página de diagnóstico de WebGPU; sirvió como desencadenante mínimo para el bloqueo de control y para la prueba comparativa en el Chromium actual.
