---
title: "Claude Desktop keeps crashing: “GPU process gone” with exit code 101457950, cause and solution"
navTitle: "Claude Desktop crash"
description: "The Claude Desktop app on Windows exits completely with “GPU process gone: exitCode 101457950” (0x060C201E), often followed by the Store app repair dialog. The full causal chain: Code Integrity blocks vk_swiftshader.dll, Chromium's fallback chain runs out, and the built-in intentional crash terminates the app. With an immediate workaround, self-diagnosis via the event log, and analysis down to the minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "9 min to read"
themen:
  - claude
slug: "the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 61bcad89e160ee37f5abd04905ed9e425236f770f9cfcc4448716acbd3569939
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:33:58.660Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump
---

The Claude Desktop app on Windows exits without an error message, all active Claude Code sessions are gone, and sometimes the app only starts again after using “Repair” in Windows Settings. The app log then contains this line:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 is hexadecimal `0x060C201E`. If you find this signature in your log, you are in the right place: This article documents the complete causal chain behind this crash, the immediate measures that make the app stable again, and the self-diagnosis that lets you confirm the finding on your own system in two minutes. Microsoft Store (MSIX) installations on all GPU vendors are affected, from Intel iGPUs to NVIDIA and AMD; the hardware, to get this out of the way, is not the cause.

## The solution in brief

The actual error lies in the app's installation package and can only be fixed by Anthropic (still open as of August 25, 2026, issue [#81341](https://github.com/anthropics/claude-code/issues/81341)). Until then, three measures make the app stable, in order of effectiveness:

**1. Enable hardware acceleration.** Check these two values in the file `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\config.json` and set them to `false` if necessary (quit the app first, then restart it):

```json
"isHardwareAccelerationDisabled": false,
"isHardwareAccelerationAutoDisabled": false
```

That sounds paradoxical because disabling hardware acceleration is usually the more stable choice. With this bug, it is the opposite, and the causal chain below explains why: The setting determines whether a GPU process crash costs only one fallback level or the entire app.

**2. Use the embedded browser sparingly.** Pages in the app's browser/preview pane trigger the crash. Closing the pane after use instead of leaving tabs parked drastically reduces the crash frequency; this correlation is documented multiple times with numbers in the community thread.

**3. Optional: Disable WebGPU.** Launching with `--disable-features=WebGPU` completely prevents the most common trigger. With a Store app, the installation path changes with every update, so use a launcher that resolves it again at every startup:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

The catch: This only works when the app is launched through this launcher. Measure 1 works on every launch.

“Repairing” or reinstalling the app does not fix the problem, by the way; it only cures the resulting symptom (more on that below). Graphics driver updates are also wasted effort.

## Self-diagnosis: confirm the finding on your own system

Two checks are sufficient. First, the crash signature in the app log:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

Second, and this is the actual proof, the Windows Code Integrity log:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

On affected systems, you will find Event 3033 entries whose timestamps match the crash times to the second, with this message:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

On the system examined here, seven out of seven crashes over three weeks matched such an event to the second, including a deliberately triggered control crash.

## The complete causal chain

The crash is the final link in a four-link chain revealed jointly by two analyses: the Code Integrity trail from community issue [#81698](https://github.com/anthropics/claude-code/issues/81698) and an independent minidump analysis ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Link 1: A page in the embedded browser needs software rendering.** A typical trigger is a WebGPU call (`navigator.gpu.requestAdapter()`), recognizable in the window log by this warning immediately before the crash:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

When the app runs without hardware acceleration, the path necessarily goes through the SwiftShader software Vulkan implementation: The GPU process tries to load the bundled `vk_swiftshader.dll`.

**Link 2: Windows Code Integrity blocks the app's own DLL.** The GPU process runs with the “MicrosoftSignedOnly” hardening policy (verifiable with `Get-ProcessMitigation`). For a Store app to be allowed to load its own vendor-signed DLLs, the MSIX package must include a signature catalog `AppxMetadata\CodeIntegrity.cat`. This exact file is missing from the shipped package, as community members demonstrated by inspecting the MSIX file. The result: Signature verification fails, Windows logs Event 3033, and forcibly terminates the GPU process. Exit code `0x060C201E` is an AppX integrity error from the Windows loader, not a Chromium code; this is why it cannot be found in any Chromium source and why the GPU process leaves no crash dump either: there is no exception to dump.

**Link 3: Chromium's fallback chain runs out.** When the GPU process crashes, Chromium falls back one rendering level: hardware GL, then software GL, then a pure display compositor. Only when no level remains does the built-in intentional crash take effect. In the source code of the bundled version (Chromium 148.0.7778.280 in Electron 42.9.2), it literally reads:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Link 4: The main process intentionally terminates itself.** This `LOG(FATAL)` is the moment when “the app crashes.” It is demonstrated by a minidump of the main process: `EXCEPTION_BREAKPOINT` (an intentional `int3`, not a driver error), not a single graphics driver DLL in the process, and in plaintext in memory:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

The fact that this dump exists at all was the hardest part of the analysis: The app's Sentry integration consumes Crashpad dumps on the next app launch, sends them to the vendor telemetry, and deletes them locally. The Crashpad folder is therefore always empty for the user. The solution is an observer independent of the app's process tree (started via WMI so the app crash does not terminate it as well), which scans the Crashpad database every 200 milliseconds for `*.dmp` and immediately copies any findings elsewhere before they are deleted. The Python package `minidump` handles the analysis, entirely without WinDbg.

## Why “disable hardware acceleration” makes everything worse

The chain also explains the most counterintuitive finding. Disabled hardware acceleration has two fatal effects here at the same time. First, it forces the SwiftShader path, meaning the exact DLL load attempt that Code Integrity blocks; with hardware acceleration enabled, `vk_swiftshader.dll` is hardly ever needed. Second, the GPU process then starts at the bottom of the fallback chain: A single crash is enough, and Link 4 takes effect. This also explains the observation from the community thread that a Code Integrity block sometimes has no consequences and sometimes terminates the app: It depends on how many fallback levels the browser process still has left.

Particularly unfortunate: The app has automatic hardware acceleration disabling after problems (`isHardwareAccelerationAutoDisabled`). Intended as a stability measure, it moves affected systems directly into the configuration in which the next crash costs the entire app.

## The resulting symptom: the repair loop

The Code Integrity failure has a side effect that many affected users mistake for a separate problem: After the incident, Windows sometimes classifies the app package as “Modified, NeedsRemediation.” The app then no longer starts at all until you reset it via Settings → Apps → Claude → Advanced options → “Repair.” Anyone who therefore “constantly has to repair the app” is seeing the same underlying problem, just one link further along: Repair fixes the package status, not the cause; the next crash follows with the next blocked DLL load attempt.

## Status of reports

The packaging cause has been reported as [#81341](https://github.com/anthropics/claude-code/issues/81341), the community evidence thread is [#81698](https://github.com/anthropics/claude-code/issues/81698), and the minidump analysis explaining the fallback chain is [#89250](https://github.com/anthropics/claude-code/issues/89250). The actual fix, a complete signature catalog in the MSIX package, is Anthropic's responsibility. Until then: Keep hardware acceleration on, close the browser pane conscientiously, and disable WebGPU with a flag if needed. On the system examined here, the app has been crash-free since implementing Measure 1.

## Sources

1.  [GitHub issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): The community evidence thread on the Code Integrity chain, cross-vendor data points, and the browser-pane correlation.

2.  [GitHub issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): The packaging cause; missing Code Integrity catalog in the MSIX.

3.  [GitHub issue #89250: Minidump analysis of app termination](https://github.com/anthropics/claude-code/issues/89250): The second half of the chain described here, including the dump capture method and fix suggestions.

4.  [Chromium source code: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): The IntentionallyCrashBrowserForUnusableGpuProcess function and fallback logic.

5.  [Electron documentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): The event an Electron app can use to observe GPU process crashes and take its own countermeasures.

6.  [Python package minidump](https://pypi.org/project/minidump/): Tool for dump analysis (exception record, module list, memory strings).

7.  [webgpureport.org](https://webgpureport.org/): WebGPU diagnostic page; used as the minimal trigger for the control crash and for comparison testing in the current Chromium.
