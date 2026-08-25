---
title: "El fallo de GPU 0x060C201E en la aplicación de escritorio Claude: una investigación hasta el minidump"
navTitle: "Fallo de GPU 0x060C201E"
description: "La aplicación de escritorio Claude se cierra de forma reproducible con «GPU process gone». Al principio todo apunta a un error del controlador AMD; después, los propios experimentos refutan la hipótesis y, al final, un minidump capturado revela la causa real: el aborto integrado de Chromium «GPU process isn't usable. Goodbye.»"
date: "2026-08-24"
kategorie: "Claude"
timeToRead: "12 min de lectura"
themen:
  - claude
slug: "el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/es/blog/el-fallo-de-gpu-0x060c201e-en-la-aplicacion-de-escritorio-claude-una-investigacion-hasta-el
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:05:58.401Z
translationReview: automatic
---

Desde finales de julio, mi aplicación de escritorio Claude se cierra varias veces al día en Windows. Sin diálogo, sin ventana de error: la aplicación simplemente desaparece, junto con todas las sesiones de Claude Code que se estaban ejecutando en ella. Ya ha ocurrido más de 25 veces. Es hora de dejar de reiniciarla y averiguar dónde se produce realmente el error. Adelanto tanto: el principal sospechoso inicial resulta no tener nada que ver, y la causa real acaba apareciendo, negro sobre blanco, en un minidump que la aplicación en realidad no quería entregar.

## La pista en el registro

La aplicación guarda sus registros en `%LOCALAPPDATA%\Claude\Logs`, mientras que las generaciones anteriores y la configuración se encuentran en la ruta virtualizada del Store `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. En `main.log` aparece exactamente lo mismo antes de cada fallo:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 en hexadecimal es `0x060C201E`. Recuerde este número: es la firma del error. El registro de la ventana aporta el desencadenante: inmediatamente antes de cada fallo, una página en el navegador integrado de la aplicación solicita un adaptador WebGPU.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

Es decir: `navigator.gpu.requestAdapter()` entra, en el proceso GPU de Chromium, en la enumeración de adaptadores de Dawn; el proceso GPU falla y, en lugar de que la aplicación lo reinicie, se cierra toda la aplicación.

## Sospechoso n.º 1: el controlador gráfico

La máquina tiene una Radeon RX 7900 XT con Adrenalin 32.0.31035.1003; la aplicación incluye Electron 42.9.2 con Chromium 148. La explicación cómoda está sobre la mesa: código Dawn antiguo se encuentra con un controlador RDNA3, el controlador falla, caso cerrado. Cómoda, plausible y, como se verá: falsa. Pero vayamos por partes, porque solo se puede refutar con experimentos.

De entrada, dos cosas quedaron descartadas como pistas falsas. La iGPU desactivada en el Administrador de dispositivos (estado «Error») es simplemente el código 22, desactivada deliberadamente. Y la aplicación ya tenía desactivada desde hacía tiempo la aceleración por hardware (`isHardwareAccelerationDisabled: true` en config.json), algo que no impresionó en absoluto a los fallos. Por qué este ajuste incluso empeora el problema solo se verá al final.

## Experimento 1: prueba de contraste en Chromium actual

Misma carga, misma máquina, navegador actual: webgpureport.org en Chromium 151 inicializa WebGPU por completo, adaptador `amd / rdna-3`, incluida la creación del dispositivo, sin ninguna anomalía. Por tanto, el controlador actual con Dawn actual funciona correctamente.

## Experimento 2: Electron 42.9.2 estándar, ruta de hardware

Si Electron 42 no se lleva bien con este controlador, debe poder reproducirse con 20 líneas. Así que: exactamente la misma versión de Electron que en la aplicación como proyecto de prueba puro, una ventana, una página, una `requestAdapter()`:

```js
const { app, BrowserWindow, crashReporter } = require('electron');
crashReporter.start({ submitURL: '', uploadToServer: false });
app.on('child-process-gone', (e, d) =>
  console.log('GONE: ' + JSON.stringify(d)));
app.whenReady().then(() => {
  const win = new BrowserWindow({ show: false });
  win.loadFile('index.html'); // ruft requestAdapter() auf
});
```

Resultado con aceleración por hardware: `adapter ok (amd/rdna-3), device ok`. Sin fallo. La ruta D3D12 de Electron 42 con este controlador funciona perfectamente. Queda así refutada la hipótesis de que «el código Dawn antiguo no tolera el controlador RDNA3».

## Experimento 3: Electron 42.9.2 estándar, ruta de software como en la aplicación

Pero la aplicación funciona sin aceleración por hardware. Por tanto, el mismo experimento con `app.disableHardwareAcceleration()`, además de un contexto WebGL activo (que en modo software funciona mediante SwiftShader) y `powerPreference: 'high-performance'` en la solicitud del adaptador, para reproducir exactamente el flujo de los registros de la aplicación:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

La misma advertencia de powerPreference que en el registro de la aplicación, la misma ruta de código hasta la enumeración de adaptadores y, después, la respuesta correcta: no hay adaptador disponible, rechazo limpio, proceso vivo. Electron 42.9.2 estándar sencillamente no falla en esta máquina, independientemente de la ruta.

## Experimento 4: otro hardware, misma firma

Antes de seguir especulando, conviene mirar el rastreador de incidencias, y allí queda claro: el fallo idéntico con el mismo código de salida 0x060C201E se ha informado varias veces, entre otros casos en una GPU para portátil NVIDIA RTX 5080. En su registro de eventos del sistema: ningún evento TDR, ningún reinicio del controlador. El controlador, sea del fabricante que sea, no es la causa. La causa del fallo está en el propio proceso GPU de la aplicación o, como se verá enseguida, en la reacción de la aplicación ante su fallo.

## Experimento 5: obtener el minidump que la aplicación borra

Hasta aquí faltaba la prueba decisiva: un volcado de fallo. La carpeta Crashpad de la aplicación estaba vacía después de cada fallo, lo que al principio parecía indicar que el informe de fallos estaba desactivado. La lista de procesos dice otra cosa: se ejecuta un proceso `crashpad-handler` cuya línea de comandos apunta a la base de datos del perfil Roaming y a una URL de carga de marcador de posición. Es el patrón habitual de la integración de Sentry en aplicaciones Electron: Crashpad escribe el volcado localmente; la biblioteca de Sentry lo consume en el siguiente inicio de la aplicación, lo envía a la telemetría del fabricante y lo borra localmente. Por tanto, los volcados existen, pero no para el usuario.

La solución es poco espectacular: un observador independiente del árbol de procesos de la aplicación (iniciado mediante WMI para que el fallo de la aplicación no se lo lleve consigo) que examina la base de datos Crashpad cada 200 milisegundos en busca de `*.dmp` y copia inmediatamente cualquier hallazgo. Después, provocar el fallo de forma deliberada: abrir webgpureport.org en el navegador integrado de la aplicación. Segundos después hay un minidump de 35 MB en la carpeta de respaldo, que Sentry intenta borrar inútilmente en el siguiente inicio de la aplicación.

## El minidump: ningún controlador a la vista

El análisis con el paquete Python `minidump` arroja tres hallazgos que cambian por completo la imagen:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

Primero: el proceso volcado no es el proceso GPU, sino el **proceso principal** de la aplicación (el PID aparece en los registros de la aplicación como `electron_main`). Segundo: la excepción es un breakpoint, es decir, un `int3` ejecutado intencionadamente. Así se cierra Chromium a sí mismo cuando se activa un `CHECK()` o `LOG(FATAL)`; un error de controlador se vería como una Access Violation. Tercero: no hay cargada ni una sola DLL de controlador gráfico en la lista de módulos del proceso.

Y en la memoria del volcado aparece en texto claro el mensaje de registro fatal:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## La resolución: el aborto integrado de Chromium

Esta línea no es un mal funcionamiento: es intencionada. En el código fuente de Chromium de la versión exactamente incluida (148.0.7778.280), en `gpu_data_manager_impl_private.cc` figura:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

La llama `FallBackToNextGpuMode()`: si el proceso GPU falla, Chromium retrocede un nivel (GL por hardware → GL por software → compositor de pantalla puro). Si la lista de modos alternativos está vacía, Chromium finaliza deliberadamente el proceso del navegador, porque sin un proceso GPU funcional ya ni siquiera puede coordinar el renderizado por software.

Esto también explica por qué la aplicación se ve mucho más afectada que un navegador normal: se inicia con la aceleración por hardware desactivada, es decir, ya en el extremo inferior de la cadena de alternativas. Si una página en el navegador integrado solicita entonces WebGPU y el proceso GPU de software falla durante ello, ya no queda ningún nivel al que Chromium pueda recurrir. La siguiente parada es «Goodbye». En un Chrome normal con aceleración por hardware activa, el mismo fallo cuesta un nivel de alternativa y el navegador sigue funcionando.

Particularmente desafortunado: la configuración de la aplicación conoce un campo `isHardwareAccelerationAutoDisabled`, por lo que aparentemente la aplicación desactiva la aceleración por hardware por sí misma después de problemas. Concebida como medida contra fallos, precisamente esto acorta la cadena de alternativas y hace que el aborto fatal sea más probable en vez de menos. Un mecanismo de protección y un interruptor de emergencia que se activan mutuamente.

## Lo que queda del código de salida

Queda el propio proceso hijo GPU, que inicia el flujo cada vez. No deja su propio informe de fallo, aunque el manejador Crashpad funciona de forma demostrable (segundos más tarde volcó el proceso principal). Esto sugiere que el proceso GPU no desencadena una excepción normal, sino que termina de forma abrupta, al estilo de `TerminateProcess`, y que de ahí procede exactamente el código de salida no documentado 0x060C201E. Su tramo final queda así en manos de Anthropic: su telemetría de Sentry recibe los volcados que se borran localmente, incluida la cuestión de si el informe de fallos cubre siquiera el proceso GPU.

## Solución provisional y estado de los informes

Dado que el desencadenante son las solicitudes WebGPU de las páginas en el navegador integrado, hasta que haya una corrección ayuda desactivar WebGPU mediante una flag de Chromium. En una instalación de Store, la ruta de instalación cambia con cada actualización; por eso un pequeño launcher la resuelve de nuevo en cada inicio:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Desde el cambio: ni un solo fallo más. El análisis completo está informado: los experimentos de laboratorio y las referencias a duplicados en la primera incidencia, la evaluación del minidump con la cadena causal en la segunda. Las tres correcciones razonables se desprenden directamente del hallazgo: aclarar la causa del fallo en el proceso GPU de software (los volcados para ello se encuentran en la telemetría del fabricante), desactivar WebGPU de forma específica tras fallos repetidos de GPU en lugar de dejar que se agote la cadena de alternativas, y replantearse la desactivación automática de la aceleración por hardware, porque acorta la cadena.

## Addendum: la solución provisional se queda corta; la solución está más abajo

Aún esa misma noche, el siguiente fallo, con firma idéntica. El motivo es sencillo: el launcher con `--disable-features=WebGPU` solo tiene efecto cuando la aplicación se inicia a través de él. Al iniciarla como de costumbre desde el menú Inicio, la aplicación se ejecuta sin la flag y, en una aplicación de Store, no hay una forma limpia de añadir flags de línea de comandos permanentemente a un inicio normal.

Pero la solución permanente lleva tiempo en la cadena causal de este artículo: el aborto fatal presupone que la cadena de alternativas esté vacía, y solo está vacía de inmediato porque la aplicación se inicia con la aceleración por hardware desactivada. Por tanto, hay que volver a activar la aceleración por hardware en la `config.json` de la aplicación:

```json
"isHardwareAccelerationDisabled": false
```

Esto tiene efecto a partir del siguiente inicio de la aplicación y resuelve ambos aspectos del problema de una vez. Primero, `requestAdapter()` se ejecuta entonces por la ruta de hardware, que en esta máquina es demostrablemente estable (experimento 2: adaptador y dispositivo sin errores). Segundo, Chromium vuelve a tener niveles alternativos en reserva: si el proceso GPU volviera a fallar, el navegador cambia al renderizado por software y sigue funcionando, en lugar de cerrarse. La desactivación original de la aceleración por hardware, probablemente establecida en algún momento como medida de estabilidad, era en realidad la condición previa del fallo.

La conclusión de la investigación: la explicación más obvia («era el controlador») habría llevado a una odisea de controladores sin resultados. Dos horas de laboratorio con la versión real del motor la refutaron, y la causa solo se encontró en el minidump que la aplicación elimina rutinariamente. Por ello, cuando un proceso GPU falla, hay cuatro comprobaciones que deben hacerse al principio antes de culpar a un fabricante: la prueba de contraste en el navegador actual, la prueba de contraste en la versión pura del motor, comprobar si otro hardware muestra la misma firma y el volcado del proceso que realmente decide el aborto.

## Fuentes

1.  [Causa raíz: «GPU process isn't usable. Goodbye.» de Chromium (GitHub-Issue #89250)](https://github.com/anthropics/claude-code/issues/89250): El análisis de minidump de este artículo como informe de error, incluido el método de captura y propuestas de corrección.

2.  [Informe inicial propio con resultados de laboratorio (GitHub-Issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Los experimentos 1 a 3 y la corrección de la hipótesis de AMD, con referencias a los duplicados.

3.  [El fallo del proceso GPU cierra toda la aplicación (GitHub-Issue #81698)](https://github.com/anthropics/claude-code/issues/81698): El mismo fallo con código de salida idéntico en una NVIDIA RTX 5080, sin eventos TDR; exculpa a los controladores gráficos.

4.  [Código fuente de Chromium: gpu_data_manager_impl_private.cc (Tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): La función IntentionallyCrashBrowserForUnusableGpuProcess y la lógica de alternativas.

5.  [Documentación de Electron: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): El evento con el que una aplicación Electron puede observar fallos del proceso GPU y adoptar sus propias medidas.

6.  [Paquete Python minidump](https://pypi.org/project/minidump/): Herramienta para analizar el volcado (registro de excepciones, lista de módulos, cadenas de memoria), sin WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): Página de diagnóstico WebGPU; sirvió como desencadenante mínimo y como prueba de contraste en Chromium actual.
