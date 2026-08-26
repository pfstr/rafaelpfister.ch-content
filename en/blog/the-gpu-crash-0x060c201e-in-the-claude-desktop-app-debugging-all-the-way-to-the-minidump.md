---
title: "Claude Desktop keeps crashing: “GPU process gone” with exit code 101457950, cause and solution"
navTitle: "Claude Desktop crash"
description: "The Claude Desktop app on Windows exits completely with “GPU process gone: exitCode 101457950” (0x060C201E), often followed by the Store app’s repair dialog. The complete causal chain: Code Integrity blocks vk_swiftshader.dll, Chromium’s fallback chain runs out, and the built-in intentional crash terminates the app. With an immediate solution, self-diagnosis via the event log, and analysis down to the minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min to read"
themen:
  - claude
slug: "the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 2cf7e9455d4d9b5c148e7b55fd0433206810dc26e53bacb85e1d2dc82a0444c6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-26T04:06:53.826Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump
---

The Claude Desktop app on Windows exits without an error message, all active Claude Code sessions are gone, and sometimes the app will only start again after using “Repair” in Windows Settings. The app log then contains this line:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 is hexadecimal `0x060C201E`. If you find this signature in your log, you are in the right place: this article documents the complete causal chain behind this crash, the immediate measures that make the app stable again, and the self-diagnosis that lets you confirm the finding on your own system in two minutes. Microsoft Store (MSIX) installations on all GPU vendors are affected, from Intel iGPUs to NVIDIA and AMD; spoiler: the hardware is not the cause.

## The solution in brief

The actual defect is in the app’s installation package and can only be fixed by Anthropic (still open as of 08/25/2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). Until then, three measures make the app stable, in order of effectiveness:

**1. Enable hardware acceleration.** Check these two values in the file `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` and set them to `false` if necessary (quit the app first, then restart it):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

That sounds paradoxical, because disabling hardware acceleration is usually the more stable choice. With this bug, it is the opposite, and the causal chain below explains why: the setting determines whether a GPU process crash costs only one fallback stage or the entire app.

**2. Use the embedded browser sparingly.** Pages in the app’s browser/preview area trigger the crash. Closing the area after use instead of leaving tabs parked drastically reduces the crash frequency; this connection is documented several times with numbers in the community thread.

**3. Optional: Disable WebGPU.** Launching with `--disable-features=WebGPU` completely prevents the most common trigger. For a Store app, the installation path changes with every update, so use a launcher that resolves it again at each startup:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

The catch: this only works if the app is launched through this launcher. Measure 1 applies at every launch.

“Repairing” or reinstalling the app does not fix the problem, incidentally; it only treats the resulting symptom (more on that below). Graphics driver updates are also wasted effort.

## Self-diagnosis: confirming the finding on your own system

Two checks are sufficient. First, the crash signature in the app log:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Second—and this is the actual proof—the Windows Code Integrity log:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

On affected systems, you will find Event 3033 entries there whose timestamps match the crash times to the second, with this message:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

On the system examined here, seven out of seven crashes over three weeks matched such an event to the second, including an intentionally triggered control crash.

## The complete causal chain

The crash is the final link in a chain of four links established jointly by two analyses: the Code Integrity trail from community issue [#81698](https://github.com/anthropics/claude-code/issues/81698) and an independent minidump analysis ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Link 1: A page in the embedded browser requires software rendering.** A typical trigger is a WebGPU call (`navigator.gpu.requestAdapter()`), identifiable in the window log by this warning immediately before the crash:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

When the app runs without hardware acceleration, it must go through the SwiftShader software Vulkan implementation: the GPU process attempts to load the bundled `vk_swiftshader.dll`.

**Link 2: Windows Code Integrity blocks the app’s own DLL.** The GPU process runs under the “MicrosoftSignedOnly” hardening policy (verifiable with `Get-ProcessMitigation`). For a Store app to load its own vendor-signed DLLs, the MSIX package must include a signature catalog `AppxMetadata\CodeIntegrity.cat`. That exact file is missing from the shipped package, as community members demonstrated by inspecting the MSIX file. The result: signature verification fails, Windows logs Event 3033, and forcibly terminates the GPU process. Exit code `0x060C201E` is an AppX integrity error from the Windows loader, not Chromium code; that is why it cannot be found in any Chromium source, and why the GPU process does not leave a crash dump either—there is no exception to dump.

**Link 3: Chromium’s fallback chain runs out.** When the GPU process crashes, Chromium falls back one rendering stage: hardware GL, then software GL, then the pure display compositor. Only when no stage remains does the built-in intentional crash take effect. In the source code of the bundled version (Chromium 148.0.7778.280 in Electron 42.9.2), it reads literally as follows:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Link 4: The main process intentionally terminates itself.** This `LOG(FATAL)` is the moment when “the app crashes.” It is demonstrated by a minidump of the main process: `EXCEPTION_BREAKPOINT` (an intentional `int3`, not a driver error), not a single graphics driver DLL in the process, and this plaintext string in memory:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

The fact that this dump exists at all was the most difficult part of the analysis: the app’s Sentry integration consumes Crashpad dumps on the next app launch, sends them to vendor telemetry, and deletes them locally. The Crashpad folder is therefore always empty for the user. The remedy is an observer independent of the app’s process tree (started via WMI so the app crash does not terminate it as well), which scans the Crashpad database every 200 milliseconds for `*.dmp` and immediately copies any findings elsewhere before they are deleted. The Python package `minidump` handles the analysis, without any need for WinDbg.

## Why “disabling hardware acceleration” makes everything worse

The chain also explains the most counterintuitive finding. Disabled hardware acceleration has two fatal effects here at once. First, it forces the SwiftShader path, meaning the exact DLL load attempt that Code Integrity blocks; with hardware acceleration enabled, `vk_swiftshader.dll` is hardly ever needed. Second, the GPU process then starts at the bottom of the fallback chain: a single crash is enough, and link 4 strikes. This also explains the observation from the community thread that a Code Integrity block sometimes has no consequence and sometimes terminates the app: it depends on how many fallback stages the browser process has left.

Particularly unfortunate: the app automatically disables hardware acceleration after problems (`isHardwareAccelerationAutoDisabled`). Intended as a stability measure, it moves affected systems directly into the configuration in which the next crash costs the entire app.

## The resulting symptom: the repair loop

The Code Integrity failure has a side effect that many affected users consider a separate problem: after the incident, Windows sometimes classifies the app package as “Modified, NeedsRemediation.” The app then no longer starts at all until it is reset through Settings → Apps → Claude → Advanced options → “Repair.” So anyone who “constantly has to repair” the app is seeing the same underlying problem, just one link further along: repair fixes the package state, not the cause; the next crash follows with the next blocked DLL load attempt.

## Status of the reports

The packaging cause has been reported as [#81341](https://github.com/anthropics/claude-code/issues/81341), the community evidence thread is [#81698](https://github.com/anthropics/claude-code/issues/81698), and the minidump analysis with the fallback-chain explanation is [#89250](https://github.com/anthropics/claude-code/issues/89250). The actual fix—a complete signature catalog in the MSIX package—is up to Anthropic. Until then: turn hardware acceleration on, close the browser area diligently, and disable WebGPU with a flag if needed. On the system examined here, the app has been crash-free since implementing measure 1.

## Sources

1.  [GitHub issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): The community evidence thread on the Code Integrity chain, cross-vendor data points, and the browser-pane correlation.

2.  [GitHub issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): The packaging cause; missing Code Integrity catalog in the MSIX.

3.  [GitHub issue #89250: Minidump analysis of the app termination](https://github.com/anthropics/claude-code/issues/89250): The second half of the chain described here, with dump capture method and proposed fixes.

4.  [Chromium source code: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): The IntentionallyCrashBrowserForUnusableGpuProcess function and fallback logic.

5.  [Electron documentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): The event an Electron app can use to monitor GPU process crashes and take its own countermeasures.

6.  [Python package minidump](https://pypi.org/project/minidump/): Tool for dump analysis (exception record, module list, memory strings).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU diagnostic page; used as the minimal trigger for the control crash and for comparison testing in the current Chromium.
