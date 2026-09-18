# Compatibility and verification status

**Audience:** users and contributors  
**Status:** living compatibility guide; checked against `Hyperdroid-recovery` main on 2026-09-18

Meldframe is still a development project. A design being present in the architecture does **not** mean it has been verified on every Android device, OEM desktop mode or runtime.

This page separates four different claims:

- **Verified** — exercised on real device/runtime evidence recorded by the implementation project;
- **Implemented, not broadly verified** — code exists, but device/OEM coverage is incomplete;
- **Experimental** — intentionally usable for validation but not a stable compatibility promise;
- **Roadmap / research** — design direction, not a current user capability.

## Device and host coverage

| Environment | Status | Evidence / limitation |
| --- | --- | --- |
| Windows Subsystem for Android (WSA) | **Primary verified development environment** | startup diagnostics, Terminal PoC, Termux `RUN_COMMAND`, Code/code-server, Wasm and selected shell-level Android task-control semantics have recorded WSA verification |
| Generic Android phone/tablet | **Partially covered by implementation; not broadly verified** | core Android behavior exists, but the current evidence set is not a representative device matrix |
| Work profile | **Implemented path, still requires real-device verification** | the implementation handover explicitly notes WSA had only user 0 and could not validate work-profile behavior |
| OEM desktop modes | **Detection/profile infrastructure exists; compatibility not broadly verified** | WSA profile is verified; Samsung/Motorola/Xiaomi/Lenovo matching must not be treated as proved control support without hardware evidence |
| Rooted Android | **Experimental provider path, not a blanket compatibility claim** | Magisk adapter exists, but root must be explicitly granted and each operation proved; root does not imply every task/window operation works |

WSA is useful engineering evidence, but it is not a substitute for Android hardware coverage. Documentation should not generalize a WSA success into “supported on Android” without qualification.

## Android task/window control

Android application launch and desktop-wide task control are separate compatibility dimensions.

| Operation/path | Current evidence |
| --- | --- |
| Normal Android app catalogue/launch coordination | **Implemented** |
| Backend-neutral task snapshot/reconciliation architecture | **Implemented** |
| Ordinary APK observing/controlling all Android tasks | **Not available by default**; Android privileged/signature boundaries apply |
| Magisk task provider | **Implemented, experimental/opt-in**; provider fails closed until uid 0 and a valid full snapshot are proved |
| WSA move/resize command semantics | **Verified at shell level on one exact WSA fingerprint** |
| WSA maximize/restore semantics | **Verified at shell level on the same fingerprint** |
| WSA close semantics | **Verified at shell level for the constrained single-task-root case** |
| WSA activate/minimize/fullscreen | **Not claimed** |
| Successful in-app Magisk control round trip | **Still open on the recorded WSA test setup**; Magisk denied Meldframe's app UID even though developer `adb shell su` access existed |
| Shizuku/system-service provider | **Roadmap/open work** |

An accepted provider command never directly changes `WindowRegistry`. A later full native snapshot must confirm the result. This prevents a timeout, partial side effect or misleading OEM command result from becoming invented shell state.

See [Android desktop integration and privileged task control](ANDROID_DESKTOP.md) for the user/tester-oriented explanation.

## Application/runtime paths

| Feature/path | Status | Notes |
| --- | --- | --- |
| Android application catalogue/launch coordination | **Implemented** | part of the current architecture |
| Meldframe internal applications | **Implemented** | used by the shell today |
| Terminal with ttyd loopback | **Verified on WSA** | development/reference transport; durable PTY reattach is not implied |
| External Termux command execution | **Verified on WSA** | readiness is probed with a real command, not package presence alone |
| code-server service-backed Code app | **Verified on WSA** | backend/service + localhost gateway + Android WebView path |
| WebAssembly probe/bundled runner | **Verified on WSA** | evidence-based capability path; not a statement that arbitrary WASI desktop apps are supported |
| Generic PRoot Linux applications | **Roadmap/partial architecture** | do not treat PRoot installation as automatic Meldframe integration |
| Wayland per-window Linux GUI | **Roadmap** | not current production functionality |
| XWayland compatibility | **Roadmap** | depends on the future Linux GUI bridge |
| AVF Linux runtime | **Research/roadmap** | no current end-user provider claim |
| SSH/remote execution | **Architecture candidate** | not a documented stable provider today |
| AppImage Installer | **Proposal** | package framework/reference-app design exists in the wiki; implementation is not claimed |
| External third-party plugin packages | **Not stable** | typed extension architecture exists; external packaging/permission lifecycle is still being designed |
| Meldframe Portal Framework | **Proposal** | desktop-service API design, not current stable SDK |

## Presentation support

| Presentation | Status | Scope |
| --- | --- | --- |
| Android task / existing Android surfaces | **Implemented** | uses Android's application/window mechanisms; privileged desktop-wide control is a separate concern |
| Meldframe-owned UI | **Implemented** | internal shell/application surfaces |
| Android WebView | **Implemented and used by verified workloads** | Terminal/service-backed Web applications |
| Wayland | **Roadmap** | future Linux GUI presentation |
| XWayland | **Roadmap** | future compatibility layer for X11 applications |
| Remote stream | **Architecture candidate** | no current stable user workflow documented |

A runtime being able to execute a process does not prove that Meldframe can present its GUI. Execution and presentation are separate compatibility dimensions.

## Build and test evidence

The implementation handover records the first successful Gradle build on 2026-09-16 and 593 green tests on 2026-09-17 (306 in `:core`, 287 in `:app`). Those numbers are historical evidence, not a permanent badge: the current branch can move beyond them, and CI/build status should be checked directly when making a release claim.

The same handover records several items that remained unverified on device, including work-profile behavior and some UI interactions. This is why this page avoids a single “supported” label for the whole application.

## How to interpret capability results

Compatibility is determined at runtime where possible. A device can legitimately report different states for different features:

```text
Termux execution      READY
WebView presentation  AVAILABLE
Android task observe  AVAILABLE_WITH_SETUP
Wayland               UNKNOWN / unavailable
work profile          not present
```

`UNKNOWN` does not mean “supported”, but it also should not be rewritten as “unsupported” without evidence. See [Capabilities](CAPABILITIES.md).

## What users should report

A useful compatibility report should include:

1. Android version, device/model and exact build fingerprint when privileged/task behavior is involved;
2. whether the environment is WSA, a normal Android device, or an OEM desktop mode;
3. the Meldframe build/commit;
4. the relevant capability/runtime/provider state;
5. the startup diagnostics and the exact failed action;
6. for Termux, whether the real command verification succeeds;
7. for privileged task control, provider name, granted operations and whether the result was confirmed by a later full snapshot;
8. whether the failure is execution, presentation, permission/setup, provider access, or UI lifecycle.

Meldframe writes startup diagnostics under the `MeldframeDiagnostics` log tag and, in the verified WSA path, to the app's external-files diagnostics file. See [Troubleshooting](TROUBLESHOOTING.md) for the current diagnostic workflow.

## What this matrix deliberately does not claim

It does not claim:

- that every Android 6+ device is functionally supported merely because `minSdk` permits installation;
- that every Termux installation is a ready runtime;
- that root enables all privileged desktop features;
- that a successful `adb shell su` test proves Meldframe's app UID has root;
- that OEM detection proves OEM task-control compatibility;
- that PRoot implies Linux GUI support;
- that Wayland/AVF/AppImage/Portal proposals are already shipped;
- that a WSA verification result proves behavior on Samsung, Lenovo, Motorola, Pixel or other OEM environments.

Those claims require evidence from the corresponding layer and device.

## Related documents

- [Features and current status](FEATURES.md)
- [Android desktop integration and privileged task control](ANDROID_DESKTOP.md)
- [Runtimes](RUNTIMES.md)
- [Capabilities](CAPABILITIES.md)
- [Installation](INSTALLATION.md)
- [Troubleshooting](TROUBLESHOOTING.md)
