---
title: "The GPU Crash 0x060C201E in the Claude Desktop App: Debugging All the Way to the Minidump"
navTitle: "GPU Crash 0x060C201E"
description: "The Claude desktop app reproducibly exits with 'GPU process gone.' At first, everything points to an AMD driver bug; then my own experiments disprove that theory, and finally an intercepted minidump reveals the actual cause: Chromium's built-in self-termination, 'GPU process isn't usable. Goodbye.'"
date: "2026-08-24"
kategorie: "Claude"
timeToRead: "12 min to read"
themen:
  - claude
slug: "the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
url: https://rafaelpfister.ch/en/blog/the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump
translationSourceHash: 6bd2b58fe661a5639010e16b417412ca9e85f687bae94531890c8fefaef4050d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:04:26.668Z
translationReview: automatic
---

Since late July, my Claude desktop app on Windows has been quitting several times a day. No dialog, no error window—the app is simply gone, along with all Claude Code sessions running in it. More than 25 times by now. Time to stop restarting it and instead find out where the error is actually occurring. Here is the short version: the prime suspect turns out to be uninvolved, and the real cause is ultimately spelled out in black and white in a minidump that the app did not actually want to give up.

## The trail in the log

The app stores its logs under `%LOCALAPPDATA%\Claude\Logs`, while older generations and the configuration are located in the virtualized Store path `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude`. In `main.log`, exactly the same thing appears before every crash:

```text
16:01:38 [info] GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
16:03:34 [info] Starting app { appVersion: '1.34493.1', ... }
```

101457950 is hexadecimal `0x060C201E`. Remember this number: it is the bug's signature. The window log provides the trigger: immediately before every crash, a page in the app's embedded browser requests a WebGPU adapter.

```text
16:01:38 [warn] The powerPreference option is currently ignored
                when calling requestAdapter() on Windows.
16:01:38 [warn] A valid external Instance reference no longer exists.
14:59:15 [warn] WebGL: CONTEXT_LOST_WEBGL: loseContext: context lost
```

So `navigator.gpu.requestAdapter()` enters Dawn's adapter enumeration in Chromium's GPU process, the GPU process crashes, and rather than restarting it, the app exits entirely.

## Suspect No. 1: the graphics driver

The machine has a Radeon RX 7900 XT with Adrenalin 32.0.31035.1003, and the app bundles Electron 42.9.2 with Chromium 148. The convenient explanation is on the table: old Dawn code meets an RDNA3 driver, the driver crashes, case closed. Convenient, plausible, and, as will become clear: wrong. But one thing at a time, because theories can only be disproven through experiments.

Two things were ruled out early as red herrings. The disabled iGPU in Device Manager (status "Error") is simply Code 22, intentionally disabled. And the app had long since disabled hardware acceleration (`isHardwareAccelerationDisabled: true` in config.json), which did nothing to impress the crashes. Why this setting actually makes the problem worse only becomes clear at the very end.

## Experiment 1: comparison in current Chromium

Same workload, same machine, current browser: webgpureport.org fully initializes WebGPU in Chromium 151, adapter `amd / rdna-3`, including device creation, without any issues whatsoever. The current driver with current Dawn is therefore fine.

## Experiment 2: stock Electron 42.9.2, hardware path

If Electron 42 cannot handle this driver, it should be reproducible in 20 lines. So: exactly the same Electron version as the app, as a pure test project, one window, one page, one `requestAdapter()`:

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

Result with hardware acceleration: `adapter ok (amd/rdna-3), device ok`. No crash. Electron 42's D3D12 path works flawlessly with this driver. That disproves the theory that "old Dawn code does not tolerate the RDNA3 driver."

## Experiment 3: stock Electron 42.9.2, software path as in the app

But the app runs without hardware acceleration. So the same experiment with `app.disableHardwareAcceleration()`, plus an active WebGL context (which runs through SwiftShader in software mode) and `powerPreference: 'high-performance'` on the adapter request, to reproduce the app logs' flow exactly:

```text
[renderer] webgl context: WebKit WebGL
[renderer] The powerPreference option is currently ignored
           when calling requestAdapter() on Windows.
[renderer] No available adapters.
[renderer] RESULT: adapter=null
TIMEOUT: no crash after 25s
```

The same powerPreference warning as in the app log, the same code path through adapter enumeration, and then the correct response: no adapter available, clean rejection, process remains alive. Stock Electron 42.9.2 simply does not crash on this machine, regardless of the path.

## Experiment 4: different hardware, same signature

Before continuing to guess, it is worth looking at the issue tracker, where it becomes clear: the identical crash with the identical exit code 0x060C201E has been reported several times, including on an NVIDIA RTX 5080 Laptop GPU. Its system event log shows no TDR events and no driver resets. The driver, regardless of manufacturer, is not the cause. The crash cause lies in the app's GPU process itself—or, as will soon become clear, in the app's response to its crash.

## Experiment 5: obtaining the minidump the app deletes

Up to this point, the crucial piece of evidence was missing: a crash dump. The app's Crashpad folder was empty after every crash, which initially looked like crash reporting had been disabled. The process list says otherwise: a `crashpad-handler` process is running, and its command line points to the database in the Roaming profile and to a placeholder upload URL. This is the usual pattern of Sentry integration in Electron apps: Crashpad writes the dump locally; the Sentry library consumes it at the next app launch, sends it to the vendor's telemetry, and deletes it locally. The dumps therefore exist, just not for the user.

The solution is unremarkable: an observer independent of the app's process tree (started through WMI so the app crash cannot take it down with it), which scans the Crashpad database every 200 milliseconds for `*.dmp` and immediately copies any findings elsewhere. Then trigger the crash deliberately: open webgpureport.org in the app's embedded browser. Seconds later, a 35 MB minidump is in the backup folder, which Sentry tries in vain to delete at the next app launch.

## The minidump: no driver anywhere in sight

Analysis with the Python package `minidump` produces three findings that completely change the picture:

```text
Exception: EXCEPTION_BREAKPOINT (0x80000003)
Adresse:   Claude.exe+0x5e8a6c9
Prozess:   PID 27660
```

First, the dumped process is not the GPU process but the app's **main process** (the PID appears in the app logs as `electron_main`). Second, the exception is a breakpoint, meaning an intentionally executed `int3`. This is how Chromium terminates itself when a `CHECK()` or `LOG(FATAL)` fires; a driver error would look like an access violation. Third, not a single graphics-driver DLL is loaded in the process's module list.

And the dump's memory contains the fatal log message in plain text:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

## The resolution: Chromium's built-in self-termination

This line is not a malfunction; it is intentional. The Chromium source code for the exact bundled version (148.0.7778.280) contains this in `gpu_data_manager_impl_private.cc`:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

It is called by `FallBackToNextGpuMode()`: when the GPU process crashes, Chromium falls back one level (hardware GL → software GL → display-only compositor). If the list of fallback modes is empty, Chromium intentionally terminates the browser process, because without a functioning GPU process it can no longer coordinate even software rendering.

This also explains why the app is hit much harder than a normal browser: it starts with hardware acceleration disabled, so it is already at the bottom of the fallback chain. If a page in the embedded browser then requests WebGPU and crashes the software GPU process in the process, Chromium has no level left to fall back to. The next stop is "Goodbye." In a normal Chrome with hardware acceleration enabled, the same crash costs one fallback level and the browser keeps running.

Particularly unfortunate: the app configuration has a field `isHardwareAccelerationAutoDisabled`, so the app apparently disables hardware acceleration itself after problems. Intended as a crash countermeasure, this does exactly the opposite: it shortens the fallback chain and makes the fatal self-termination more likely rather than less likely. A protective mechanism and an emergency shutoff that arm each other.

## What remains of the exit code

That leaves the GPU child process itself, which initiates the sequence each time. It leaves no crash report of its own, even though the Crashpad handler demonstrably works (it dumped the main process seconds later). This suggests that the GPU process does not trigger a normal exception, but is terminated forcefully, in the style of `TerminateProcess`, and that the undocumented exit code 0x060C201E comes from exactly that. Its final mile therefore lies with Anthropic: its Sentry telemetry receives the dumps that are deleted locally, including the question of whether crash reporting covers the GPU process at all.

## Workaround and status of the reports

Because the trigger is the pages' WebGPU requests in the embedded browser, disabling WebGPU via a Chromium flag helps until a fix is available. With a Store installation, the installation path changes with every update, so a small launcher resolves it again at every start:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

Since making the change: not a single crash. The full analysis has been reported: the lab experiments and duplicate references in the first issue, the minidump analysis with the causal chain in the second. The three sensible fixes follow directly from the findings: determine the crash cause in the software GPU process (the relevant dumps are in the vendor telemetry), disable WebGPU specifically after repeated GPU crashes rather than allowing the fallback chain to run out, and reconsider automatic hardware-acceleration disabling because it shortens the chain.

## Addendum: the workaround falls short; the solution runs deeper

That very same evening, the next crash occurred, with the identical signature. The reason is simple: the launcher with `--disable-features=WebGPU` only works when the app is launched through it. When starting it the usual way from the Start menu, the app runs without the flag, and with a Store app there is no clean way to permanently supply command-line flags to a normal launch.

But the permanent solution has long been present in this article's causal chain: the fatal self-termination requires the fallback chain to be empty, and it is only empty immediately because the app starts with hardware acceleration disabled. So hardware acceleration needs to be enabled again in the app's `config.json`:

```json
"isHardwareAccelerationDisabled": false
```

This takes effect at the next app launch and fixes both sides of the problem at once. First, `requestAdapter()` then runs through the hardware path, which is demonstrably stable on this machine (Experiment 2: adapter and device without errors). Second, Chromium once again has fallback levels in reserve: if the GPU process does crash again, the browser falls back to software rendering and keeps running instead of terminating. Disabling hardware acceleration originally—probably set at some point as a stability measure—was actually the prerequisite for the crash.

The conclusion of the investigation: the most obvious explanation ("it was the driver") would have led to a fruitless driver odyssey. Two hours of lab work with the real engine version disproved it, and the cause only emerged in the minidump that the app routinely clears away. When a GPU process crashes, four checks should therefore come first before blaming a vendor: a comparison in the current browser, a comparison in the pure engine version, a look at whether other hardware shows the same signature, and a dump of the process that actually decides to terminate.

## Sources

1.  [Root cause: Chromium's 'GPU process isn't usable. Goodbye.' (GitHub issue #89250)](https://github.com/anthropics/claude-code/issues/89250): This article's minidump analysis as a bug report, including the capture method and proposed fixes.

2.  [My initial report with lab results (GitHub issue #89226)](https://github.com/anthropics/claude-code/issues/89226): Experiments 1 through 3 and the correction of the AMD theory, with references to the duplicates.

3.  [GPU process crash kills entire app (GitHub issue #81698)](https://github.com/anthropics/claude-code/issues/81698): The same crash with the identical exit code on an NVIDIA RTX 5080, without TDR events; exonerates the graphics drivers.

4.  [Chromium source code: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): The IntentionallyCrashBrowserForUnusableGpuProcess function and fallback logic.

5.  [Electron documentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): The event through which an Electron app can observe GPU process crashes and take its own countermeasures.

6.  [Python package minidump](https://pypi.org/project/minidump/): Dump-analysis tool (exception record, module list, memory strings), entirely without WinDbg.

7.  [webgpureport.org](https://webgpureport.org/): WebGPU diagnostic page; used as the minimal trigger and for comparison in current Chromium.
