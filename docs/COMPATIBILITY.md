# Compatibility and verification status

**Audience:** users and contributors  
**Status:** living compatibility guide; checked against `Hyperdroid-recovery` main on 2026-09-19

Meldframe is still a development project. This page records evidence, not aspirations. The implementation repository is the source of truth when this page and code disagree.

Use these labels narrowly:

- **Verified** — exercised in a recorded device/runtime test.
- **Implemented, not broadly verified** — code exists, but the evidence does not justify a general device claim.
- **Experimental** — deliberately exposed for validation; not a stable compatibility promise.
- **Roadmap / research** — design or prototype work, not a current user capability.

## Device and host coverage

| Environment | Status | Evidence / limitation |
| --- | --- | --- |
| Windows Subsystem for Android (WSA) | **Primary verified development environment** | startup diagnostics, Terminal/Termux, Code/code-server, WebAssembly and selected task-control behavior have recorded verification |
| Generic Android phone/tablet | **Implemented in part; not broadly verified** | the current evidence set is not a representative device matrix |
| Work profile | **Implemented path; real-device verification still required** | recorded WSA testing had only user 0 |
| OEM desktop modes | **Detection/profile infrastructure exists; compatibility not broadly verified** | detecting Samsung/Motorola/Xiaomi/Lenovo does not prove their task/window control behavior |
| Rooted Android | **Experimental provider path** | Magisk integration exists, but root must be explicitly granted and each operation independently proved |

WSA evidence must not be generalized into a blanket “supported on Android” claim.

## Android task and window control

Android application launch and desktop-wide task control are separate compatibility dimensions.

| Operation/path | Current evidence |
| --- | --- |
| Android app catalogue and launch coordination | **Implemented** |
| Backend-neutral task snapshot/reconciliation | **Implemented** |
| Ordinary APK observing/controlling every Android task | **Not available by default**; Android privileged/signature boundaries apply |
| Magisk task provider | **Implemented, experimental/opt-in**; fails closed until uid 0 and a valid full snapshot are proved |
| WSA move/resize command semantics | **Verified at shell level on one exact WSA fingerprint** |
| WSA maximize/restore semantics | **Verified at shell level on the same fingerprint** |
| WSA close semantics | **Verified at shell level for the constrained single-task-root case** |
| WSA activate/minimize/fullscreen | **Not claimed** |
| Successful in-app Magisk control round trip | **Still open in the recorded WSA setup**; developer `adb shell su` access did not imply root for Meldframe's app UID |
| Shizuku/system-service provider | **Roadmap/open work** |

An accepted provider command does not directly mutate `WindowRegistry`. A later full native snapshot must confirm the result. See [Android desktop integration and privileged task control](ANDROID_DESKTOP.md).

## Application and runtime paths

| Feature/path | Status | Notes |
| --- | --- | --- |
| Meldframe internal applications | **Implemented** | current shell surfaces |
| Internal Browser, including multiple tabs | **Implemented; WSA-verified behavior** | not the proposed Browser Broker and not a Chromium-extension/automation API |
| Terminal with ttyd loopback | **Verified on WSA** | reference transport; durable PTY reattach is not implied |
| External Termux command execution | **Verified on WSA** | readiness requires setup and a real command round trip, not package presence alone |
| code-server-backed Code app | **Verified on WSA** | service + localhost gateway + Android WebView path |
| WebAssembly probe/bundled runner | **Verified on WSA** | does not imply arbitrary WASI desktop-app compatibility |
| Linux `.desktop` parsing/discovery | **Implemented and tested with real application entries** | WPS Office 12.1.2 and Cylheim 4.11.1 entries were used as real-file fixtures/evidence |
| Registered Linux applications in Start | **Implemented and WSA-verified** | namespaced by container; localized names work; current icon is a generic placeholder |
| Launching `AppId.Linux` GUI applications from Start | **Not implemented** | discovery/listing is deliberately separate from execution/presentation |
| Guest capability probe model/verdict logic | **Implemented and unit-tested, not provider-wired** | reports `PRESENT` / `ABSENT` / `UNKNOWN`; no production provider currently invokes it |
| Generic PRoot Linux GUI integration | **Not a current user capability** | PRoot is a compatibility mechanism, not a presentation backend or security boundary |
| Wayland per-window Linux GUI | **Roadmap** | no production presentation backend |
| XWayland compatibility | **Roadmap** | XWayland alone is insufficient without a Wayland compositor/presentation path |
| AVF Linux runtime | **Research/roadmap** | no current end-user provider claim |
| SSH/remote execution | **Architecture candidate** | no documented stable provider/user workflow today |
| External third-party plugin packages | **Not stable** | typed extension architecture exists; packaging/permission lifecycle remains under design |
| Meldframe Portal Framework | **Proposal** | desktop-service API design, not a stable SDK |

### Linux GUI research evidence

The M3 whole-display experiment has crossed some boundaries but not the product boundary. A real Cylheim Linux GUI application ran under nested sway; the compositor rendered it; and an RFB probe decoded the complete frame from wayvnc. noVNC did not paint that stream, and the same blank result reproduced outside Meldframe in headless Edge.

Therefore the defensible claim is **application → compositor → RFB wire demonstrated in the research harness**. It is not evidence that Meldframe currently launches Linux GUI applications, ships a working whole-display client, or provides per-window Wayland integration. WSL was used as the experiment runtime because the development WSA guest lacked network access; this does not make WSL a supported Meldframe RuntimeProvider.

## Presentation support

| Presentation | Status | Scope |
| --- | --- | --- |
| Android task / Android surfaces | **Implemented** | privileged desktop-wide control remains a separate concern |
| Meldframe-owned UI | **Implemented** | internal shell/application surfaces |
| Android WebView | **Implemented and used by verified workloads** | Browser and service-backed Web applications |
| Linux whole-display RFB path | **Research PoC only; partial chain demonstrated** | frame reached RFB wire; noVNC presentation remained unresolved in the harness |
| Wayland | **Roadmap** | future Linux GUI presentation |
| XWayland | **Roadmap** | future X11 compatibility layer |
| Remote stream | **Architecture candidate** | no stable user workflow documented |

Execution, discovery and presentation are independent. A process running does not prove Meldframe can display it; a `.desktop` entry appearing in Start does not prove Meldframe can launch its GUI.

## Capability-probe caveats

`GuestCapabilityProbe` is a useful example of why compatibility cannot be reduced to “Termux installed”. Native Termux and Debian under PRoot can differ in libc, compositor/socket availability and graphics capabilities even when both live behind the same Android app.

Two verdict rules are particularly important:

- musl can make `ldd --version` exit successfully, so exit code 0 is not proof of glibc; the implementation judges libc from output;
- `UNKNOWN` is not `ABSENT`, but it is also not sufficient evidence to select a presentation backend.

The probe is currently a tested core primitive, not a live compatibility scanner exposed through production providers or the planner.

## Build and test evidence

Recorded implementation milestones include a first successful Gradle build on 2026-09-16 and later test counts that increased as the Linux discovery work landed (638 tests were reported with the Start-listing milestone). These are historical snapshots, not release badges. Check the current implementation branch and CI before making a current build-health claim.

## How to interpret capability results

Where runtime capability states are available, treat them per feature rather than as one device-wide support flag:

```text
Termux execution      READY
WebView presentation  AVAILABLE
Android task observe  AVAILABLE_WITH_SETUP
Wayland               UNKNOWN / unavailable
work profile          not present
```

`UNKNOWN` does not mean supported and should not be rewritten as unsupported without evidence. See [Capabilities](CAPABILITIES.md).

## What users should report

A useful compatibility report includes:

1. Android version, device/model and exact build fingerprint when privileged/task behavior is involved;
2. whether the environment is WSA, ordinary Android, or an OEM desktop mode;
3. the Meldframe build/commit;
4. the relevant capability/runtime/provider state;
5. startup diagnostics and the exact failed action;
6. for Termux, whether the real command verification succeeds;
7. for Linux discovery, the registered container/path and whether the entry is absent from discovery, absent from Start, or merely cannot launch;
8. for privileged task control, provider name, granted operations and whether a later full snapshot confirmed the result;
9. whether the failure is discovery, execution, presentation, permission/setup, provider access, or UI lifecycle.

Meldframe writes startup diagnostics under the `MeldframeDiagnostics` log tag and, in the verified WSA path, to the app's external-files diagnostics file. See [Troubleshooting](TROUBLESHOOTING.md).

## Claims this matrix deliberately does not make

This page does **not** claim that:

- every Android device allowed by `minSdk` is functionally supported;
- every Termux installation is a ready runtime;
- root enables every privileged desktop feature;
- `adb shell su` proves Meldframe's app UID has root;
- OEM detection proves OEM task-control compatibility;
- PRoot implies isolation or Linux GUI support;
- Linux application discovery or Start listing implies Linux GUI launch support;
- the M3 RFB experiment is production Wayland support;
- the existence of `GuestCapabilityProbe` means providers already use it;
- Wayland, XWayland, AVF, remote execution or Portal proposals are shipped;
- one WSA verification proves behavior on Samsung, Lenovo, Motorola, Pixel or another OEM environment.

Each claim requires evidence from the corresponding layer and device.

## Related documents

- [Features and current status](FEATURES.md)
- [Linux applications](LINUX_APPS.md)
- [Android desktop integration and privileged task control](ANDROID_DESKTOP.md)
- [Runtimes](RUNTIMES.md)
- [Capabilities](CAPABILITIES.md)
- [Installation](INSTALLATION.md)
- [Troubleshooting](TROUBLESHOOTING.md)
