---
title: "Claude Desktop keeps crashing: “GPU process gone” with exit code 101457950, cause and solution"
navTitle: "Claude Desktop crash"
description: "The Claude Desktop app on Windows exits completely with “GPU process gone: exitCode 101457950” (0x060C201E), often followed by the Store app repair dialog. The full causal chain: Code Integrity blocks vk_swiftshader.dll, Chromium’s fallback chain runs out, and the built-in intentional crash terminates the app. With a permanent solution (switching to the classic installation without MSIX), self-diagnosis via the event log, and analysis down to the minidump."
date: "2026-08-25"
kategorie: "Claude"
timeToRead: "10 min to read"
themen:
  - claude
slug: "the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump"
translationId: "article-0932cd50b8160b45"
translationOf: claude-desktop-webgpu-absturz
translationSourceHash: 9bd177698f756b588c97f11ea4c73ea48efcfb96c8dbe1435f5ff9fc11dffec0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:27:48.832Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/the-gpu-crash-0x060c201e-in-the-claude-desktop-app-debugging-all-the-way-to-the-minidump
---

The Claude Desktop app on Windows exits without an error message, all running Claude Code sessions are gone, and sometimes the app only starts again after a “Repair” through Windows Settings. This line then appears in the app log:

```text
GPU process gone: {
  type: 'GPU',
  reason: 'crashed',
  exitCode: 101457950,
  serviceName: 'GPU'
}
```

101457950 is `0x060C201E` in hexadecimal. If you find this signature in your log, you are in the right place: this article documents the complete causal chain behind this crash, the immediate measures that make the app stable again, and the self-diagnosis that lets you confirm the finding on your own system in two minutes. MSIX installations (from the Microsoft Store or via MSIX setup) are affected across all GPU vendors, from Intel integrated GPUs to NVIDIA and AMD; the hardware is not the cause. The classic installation without MSIX is unaffected, and that is precisely the solution.

## The solution in brief: switch to the classic installation

The actual error is in the MSIX installation package and can only be fixed by Anthropic (issue [#81341](https://github.com/anthropics/claude-code/issues/81341) remains open as of October 25, 2026; reports in the issue in early September 2026 also confirmed the error in version 1.40609.1). However, the same app is also available as a classic installation without MSIX, and it is not subject to the AppX signature check that terminates the GPU process. Switching is therefore the only measure that fully eliminates the crash; it has been confirmed both in issue [#81341](https://github.com/anthropics/claude-code/issues/81341) and on the system examined here. The feature set is identical, and the update feed delivers the same versions for both variants.

**Step 1: Download and run the classic installer.** The download at [claude.com/download](https://claude.com/download) provides a Squirrel installer that installs the app to `%LOCALAPPDATA%\AnthropicClaude` (no administrator rights required). From the command line:

```powershell
curl.exe -L -o "$env:USERPROFILE\Downloads\Claude-Setup-x64.exe" `
  "https://storage.googleapis.com/osprey-downloads-c02f6a0d-347c-492b-a752-3e0651722e97/nest-win-x64/Claude-Setup-x64.exe"
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-L` | follows HTTP redirects to the actual file |
| `-o <pfad>` | target file; here, the Downloads folder |
| `<url>` | official installer source; identical to the target of the download redirect from claude.ai |

</details>

After downloading, verify the signature (`Get-AuthenticodeSignature`, expected: `Valid`, issuer “Anthropic, PBC”) and run the file. The installer initially installs an older base version; the update mechanism brings it up to date, either automatically on first launch or immediately via:

```powershell
& "$env:LOCALAPPDATA\AnthropicClaude\Update.exe" `
  --update https://downloads.claude.ai/releases/win32/x64
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `--update <url>` | downloads the latest version from the specified release feed and installs it as a new `app-<version>` directory |

</details>

**Step 2: Transfer the configuration.** The MSIX version stores login data, MCP server configuration, and settings in its virtualized container; the classic app reads `%APPDATA%\Claude`. Copy them once (close the MSIX app first; both variants cannot run simultaneously anyway because they share a single-instance lock):

```powershell
robocopy "$env:LOCALAPPDATA\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude" `
  "$env:APPDATA\Claude" /E /XD Cache "Code Cache" GPUCache claude-code Crashpad logs sentry
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `<quelle>` | configuration folder in the virtualized AppData of the MSIX package |
| `<ziel>` | configuration folder of the classic installation |
| `/E` | copies all subdirectories, including empty ones |
| `/XD <namen>` | skips the specified directories; here, caches and runtime data that the new app creates itself |

</details>

Chat histories are not lost: they are stored in the claude.ai account or (for Claude Code sessions) under `%USERPROFILE%\.claude` and are not tied to the app installation.

**Step 3: Remove the MSIX package.** Otherwise, old shortcuts will continue to launch the crashing variant:

```powershell
Get-AppxPackage Claude | Remove-AppxPackage
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `Claude` | positional name argument for `Get-AppxPackage`; filters installed AppX/MSIX packages by package name (wildcards allowed) |
| `Remove-AppxPackage` | removes the package passed through the pipeline for the current user account |

</details>

The Start menu entry “Anthropic → Claude” then belongs to the classic installation; any taskbar pin must be set again.

## If you must stay with the MSIX package

Without switching, only measures that reduce the crash frequency remain; they do not eliminate the cause:

**Use the embedded browser sparingly.** Pages in the app’s browser/preview pane trigger the crash. Closing the pane after use instead of leaving tabs parked noticeably reduces crash frequency; this relationship has been documented repeatedly with numbers in the community thread.

**Disable WebGPU.** Launching with `--disable-features=WebGPU` prevents the most common trigger. With an MSIX package, the installation path changes with every update, so use a launcher that resolves it again on every launch:

```bat
@echo off
for /f "delims=" %%i in ('powershell -NoProfile -Command ^
  "(Get-AppxPackage Claude).InstallLocation"') do set PKG=%%i
start "" "%PKG%\app\Claude.exe" --disable-features=WebGPU
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `for /f "delims="` | processes command output line by line; empty `delims=` takes the entire line (including spaces in the path) into `%%i` |
| `-NoProfile` | launches PowerShell without profile scripts, for fast and reproducible startup |
| `-Command` | executes the specified expression; `(Get-AppxPackage Claude).InstallLocation` returns the package’s current installation path |
| `start ""` | starts the program detached from the batch window; the empty quotation marks are the (empty here) window title |
| `--disable-features=WebGPU` | Chromium switch: disables the specified feature, here the WebGPU API |

</details>

This only works if the app is launched through this launcher.

The first version of this article recommended, as its primary measure, enabling hardware acceleration through `isHardwareAccelerationDisabled: false` in `config.json`. That recommendation is outdated: in current versions (1.37937.x), the flag no longer exists, the app starts with hardware acceleration enabled by default, and it still crashes (details in the addendum below).

Incidentally, “Repairing” or reinstalling the MSIX package does not fix the problem; it only treats the secondary symptom (more on that below). Graphics driver updates are also futile.

## Self-diagnosis: confirm the finding on your own system

Two checks are sufficient. First, the crash signature in the app log:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Claude\Logs\main.log" `
  -Pattern 'GPU process gone'
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-Path` | file to search, here the app’s main log |
| `-Pattern` | search pattern (regular expression); outputs all lines containing the crash signature |

</details>

Second, and this is the actual proof, the Windows CodeIntegrity log:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033
} -MaxEvents 30 | Where-Object { $_.Message -match 'claude' } |
  Select-Object TimeCreated, Message
```

<details class="options-details">
<summary>Options explained</summary>

| Option | Effect |
|---|---|
| `-FilterHashtable` | filters during retrieval: `LogName` names the event log, `Id` the event ID 3033 (Code Integrity block) |
| `-MaxEvents 30` | limits the query to the 30 most recent matches |
| `Where-Object { … -match 'claude' }` | retains only events whose message text contains the app path |
| `Select-Object TimeCreated, Message` | reduces the output to timestamp and message for comparison with crash times |

</details>

On affected systems, you will find Event 3033 entries whose timestamps match the crash times to the second, with this message:

```text
Code Integrity determined that a process
(...\WindowsApps\Claude_..._x64__pzs8sxrjxfjjc\app\claude.exe)
attempted to load ...\app\vk_swiftshader.dll that did not meet
the Microsoft signing level requirements.
```

On the system examined here, seven out of seven crashes over three weeks matched such an event to the second, including one deliberately triggered control crash.

## The complete causal chain

The crash is the final link in a four-link chain established by two analyses: the Code Integrity trail from community issue [#81698](https://github.com/anthropics/claude-code/issues/81698) and an independent minidump analysis ([#89250](https://github.com/anthropics/claude-code/issues/89250)).

**Link 1: A page in the embedded browser requires software rendering.** A typical trigger is a WebGPU call (`navigator.gpu.requestAdapter()`), recognizable in the window log by this warning immediately before the crash:

```text
[warn] The powerPreference option is currently ignored
       when calling requestAdapter() on Windows.
```

When the app runs without hardware acceleration, the path must go through the SwiftShader software Vulkan implementation: the GPU process attempts to load the bundled `vk_swiftshader.dll`.

**Link 2: Windows Code Integrity blocks the app’s own DLL.** The GPU process runs under the “MicrosoftSignedOnly” hardening policy (verifiable with `Get-ProcessMitigation`). For a Store app to load its own vendor-signed DLLs, the MSIX package must include a signature catalog `AppxMetadata\CodeIntegrity.cat`. This file is missing from the delivered package, as community members demonstrated by inspecting the MSIX file. The result: signature validation fails, Windows logs Event 3033, and it forcefully terminates the GPU process. Exit code `0x060C201E` is an AppX integrity error from the Windows loader, not a Chromium code; this is why it cannot be found in any Chromium source and why the GPU process leaves no crash dump—there is no exception to dump.

**Link 3: Chromium’s fallback chain runs out.** When the GPU process crashes, Chromium falls back one rendering level: hardware GL, then software GL, then the pure display compositor. Only when no level remains does the built-in intentional crash take effect. In the source code of the bundled version (Chromium 148.0.7778.280 in Electron 42.9.2), it reads verbatim:

```cpp
NOINLINE void IntentionallyCrashBrowserForUnusableGpuProcess() {
  LOG(FATAL) << "GPU process isn't usable. Goodbye.";
}
```

**Link 4: The main process exits deliberately.** This `LOG(FATAL)` is the moment when “the app crashes.” It is demonstrated by a minidump of the main process: `EXCEPTION_BREAKPOINT` (an intentional `int3`, not a driver error), not a single graphics driver DLL in the process, and this text in cleartext in memory:

```text
FATAL:content\browser\gpu\gpu_data_manager_impl_private.cc:418]
GPU process isn't usable. Goodbye.
```

The fact that this dump exists at all was the most difficult part of the analysis: the app’s Sentry integration consumes Crashpad dumps on the next app launch, sends them to vendor telemetry, and deletes them locally. The Crashpad folder is therefore always empty for the user. The remedy is an observer independent of the app’s process tree (started through WMI so the app crash does not terminate it too) that scans the Crashpad database every 200 milliseconds for `*.dmp` and immediately copies findings elsewhere before they are deleted. The Python package `minidump` handles the analysis, entirely without WinDbg.

## Why “disable hardware acceleration” makes everything worse

The chain also explains the most counterintuitive finding. Disabled hardware acceleration has two fatal effects here at once. First, it forces the SwiftShader path, which means precisely the DLL loading attempt that Code Integrity blocks; with hardware acceleration enabled, `vk_swiftshader.dll` is hardly ever needed. Second, the GPU process then starts at the bottom of the fallback chain: a single crash is enough, and link 4 takes effect. This also explains the observation in the community thread that a Code Integrity block is sometimes harmless and sometimes terminates the app: it depends on how many fallback levels remain for the browser process.

Especially unfortunate: the app had automatic hardware acceleration disabling after problems (`isHardwareAccelerationAutoDisabled`). Intended as a stability measure, it moved affected systems directly into the configuration in which the next crash costs the entire app.

## Addendum, August 27, 2026: hardware acceleration alone is not enough

The first version of this article recommended active hardware acceleration as the most effective immediate measure, and for two days the app did indeed remain crash-free. Then came the auto-update to 1.37937.3, and with it three crashes in one afternoon, each with the familiar Event 3033 for `vk_swiftshader.dll`. Two findings follow:

First, the missing signature catalog is also absent from the current MSIX package; the underlying problem remains unchanged in 1.37937.3.

Second, active hardware acceleration provides only statistical protection: it lengthens the fallback chain, but does not prevent Chromium from traversing it to the SwiftShader level under load or after a hardware GPU process error. Once that happens, Code Integrity blocks the DLL and the chain can still run out. In addition, the configuration flags `isHardwareAccelerationDisabled`/`isHardwareAccelerationAutoDisabled` disappeared from `config.json` in 1.37937.x; the setting can no longer be fixed there at all.

That left only the switch to the classic installation described above as a reliable solution. Since switching on the system examined here: the same app version, identical usage including the browser pane, not a single Event 3033, and no crash.

## The secondary symptom: the repair loop

The Code Integrity failure has a side effect that many affected users consider a separate problem: Windows sometimes classifies the app package as “Modified, NeedsRemediation” after the incident. The app then no longer starts at all until you reset it through Settings → Apps → Claude → Advanced options → “Repair.” Anyone who “constantly has to repair” the app is therefore seeing the same underlying problem, just one link later: repair fixes the package status, not the cause; the next crash follows on the next blocked DLL loading attempt.

## Status of reports

The packaging cause has been reported as [#81341](https://github.com/anthropics/claude-code/issues/81341), the collection thread with the community evidence is [#81698](https://github.com/anthropics/claude-code/issues/81698), the minidump analysis with the fallback-chain explanation is [#89250](https://github.com/anthropics/claude-code/issues/89250), and another detailed report including the repair loop is [#80444](https://github.com/anthropics/claude-code/issues/80444). The actual fix—a complete signature catalog in the MSIX package—is Anthropic’s responsibility. As of October 25, 2026, #81341 remains open, and reports from early September 2026 confirm the error in version 1.40609.1 as well. #80444 was closed on September 15, 2026 without specifying a fixed version, and #89250 was automatically closed for inactivity on October 20, 2026; neither issue mentions a fix. Until then: switch to the classic installation; those who must stay with the MSIX package should diligently close the browser pane and disable WebGPU by flag if needed. On the system examined here, the app has been crash-free since switching to the classic installation, without a single additional Event 3033.

## Change log

- **October 25, 2026:** Added: issue #81341 remains open as of October 25, 2026, and the error is also confirmed in version 1.40609.1. Issues #80444 and #89250 were closed without naming a fix.

## Sources

1.  [GitHub issue #81698: GPU process crash kills entire app](https://github.com/anthropics/claude-code/issues/81698): The collection thread containing community evidence for the Code Integrity chain, cross-vendor data points, and browser-pane correlation.

2.  [GitHub issue #81341: CIG + vk_swiftshader.dll kills GPU process](https://github.com/anthropics/claude-code/issues/81341): The packaging cause; missing CodeIntegrity catalog in the MSIX.

3.  [GitHub issue #89250: Minidump analysis of the app termination](https://github.com/anthropics/claude-code/issues/89250): The second half of the chain described here, including the dump-capture method and fix suggestions.

4.  [GitHub issue #80444: GPU crash with forensics and repair loop](https://github.com/anthropics/claude-code/issues/80444): Detailed individual report with timelines, event log analysis, and the finding that every crash puts the package into the “Modified” state.

5.  [Claude Desktop: official download page](https://claude.com/download): Source for the classic Windows installer (x64 and ARM64).

6.  [Chromium source code: gpu_data_manager_impl_private.cc (tag 148.0.7778.280)](https://chromium.googlesource.com/chromium/src/+/refs/tags/148.0.7778.280/content/browser/gpu/gpu_data_manager_impl_private.cc): The IntentionallyCrashBrowserForUnusableGpuProcess function and the fallback logic.

7.  [Electron documentation: child-process-gone](https://www.electronjs.org/docs/latest/api/app#event-child-process-gone): The event through which an Electron app can observe GPU process crashes and take its own countermeasures.

8.  [Python package minidump](https://pypi.org/project/minidump/): Tool used for dump analysis (exception record, module list, memory strings).

9.  [webgpureport.org](https://webgpureport.org/): WebGPU diagnostic page; used as a minimal trigger for the control crash and for the comparison test in current Chromium.
