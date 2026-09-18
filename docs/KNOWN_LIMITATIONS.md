# Known limitations

**Audience:** users, testers and evaluators  
**Status:** user documentation for current development builds

This page is the short version of what Meldframe **does not yet prove or provide**. It is intentionally conservative. A design in the roadmap, an interface in source code, or a successful shell experiment is not enough to call a feature generally supported.

For positive feature descriptions, see [Features and current status](FEATURES.md). For device evidence, see [Compatibility and verification](COMPATIBILITY.md).

## Device coverage is narrow

The strongest recorded end-to-end device evidence is currently from **Windows Subsystem for Android (WSA)**. That is useful architecture evidence, but it is not a representative Android/OEM compatibility matrix.

In particular:

- Samsung, Motorola, Xiaomi and Lenovo desktop-mode/profile detection exists in code or profile data, but representative hardware verification is still required;
- work-profile behavior is represented in the application identity/catalogue architecture, but the recorded WSA environment does not provide representative work-profile validation;
- success on one WSA build must not be interpreted as proof that privileged task-control commands work on arbitrary Android releases or OEM builds.

## Android window control is not generally available

Meldframe can launch Android applications through its common application pipeline. Desktop-wide observation and control of arbitrary Android tasks is a separate, more privileged problem.

The current architecture has an operation-scoped task-provider boundary and an opt-in Magisk adapter. Recorded WSA experiments establish selected **shell-level** semantics for move/resize, close and maximize/restore on a specific build.

They do **not** establish a generally working in-app privileged backend:

- the recorded WSA device still denies root to Meldframe's application UID;
- activate and minimize are not established through that provider path;
- Shizuku/system-service providers remain open work;
- custom decorations, per-task density and full replacement of OEM desktop presentation are later stages, not current guarantees.

See [Android desktop integration](ANDROID_DESKTOP.md).

## Linux GUI integration is not complete

The current verified Linux/runtime path is centered on command execution, Terminal and service-backed Web applications.

A production-quality Linux GUI stack is still roadmap work. In particular, do not assume current builds provide:

- general Wayland application discovery and launch;
- independent `xdg_toplevel` → Meldframe window mapping;
- XWayland compatibility;
- robust clipboard/drag-and-drop/IME integration for arbitrary Linux GUI applications;
- dma-buf/AHardwareBuffer/SurfaceControl zero-copy presentation;
- GPU/Vulkan/Turnip compatibility across devices.

Wayland-related architecture and research should be read as future integration design unless a specific document says otherwise.

## Termux is the current verified runtime path, not the final runtime model

The first real `RuntimeProvider` uses external Termux and verifies readiness with a real command round-trip. Current setup requires Termux external-command support and the corresponding permission.

Limitations include:

- generic PRoot distro discovery/entry is not yet the completed provider path;
- AVF-backed Debian/Linux guests are roadmap work;
- SSH/remote providers are architectural candidates, not current first-party support;
- environment handling, cancellation and fully shaped streaming process APIs remain incomplete;
- PRoot, when added, must be treated as a compatibility mechanism rather than a security boundary.

See [Runtimes](RUNTIMES.md).

## Terminal is a validated PoC, not yet a mature terminal emulator product

The Terminal validates xterm.js/WebView presentation, a byte-oriented protocol, runtime targeting and a real ttyd transport. Current limitations include incomplete product UI around tabs/panes/profiles, reconnect semantics, input calibration and physical-keyboard/IME edge cases.

The ttyd path is a transport implementation. It is not the long-term Terminal domain model, and current ttyd sessions should not be assumed to provide robust detach/reattach semantics.

See [Terminal](TERMINAL.md).

## Code proves service-backed composition, not universal Linux desktop compatibility

Code/code-server demonstrates that a Linux service can be supervised and presented through a Meldframe-owned Android WebView with a local gateway.

That proves the service-backed application pattern. It does not prove that arbitrary Electron, VS Code desktop or Linux GUI applications can already run as native Meldframe windows.

See [Code / service-backed apps](CODE.md).

## Guest Protocol is not a stable public protocol yet

The roadmap defines a future versioned host↔guest boundary for exec, PTY, services, files and desktop integration. Current Termux intents, ttyd and code-server paths provide implementation evidence for that design, but they are not themselves a completed general Guest Protocol.

There is currently no stable public `meldframe-agent`/Guest Protocol SDK that third parties should depend on.

See [Guest Protocol](GUEST_PROTOCOL.md).

## External plugins are not a stable distribution surface yet

Typed extension contributions exist and built-in extensions exercise several contribution kinds. The public third-party packaging, process boundary, permission model, signing/update story and compatibility policy are not frozen.

Do not publish an external plugin expecting a stable ABI/API based only on current internal extension interfaces.

See [Extensions and plugins](EXTENSIONS.md).

## Portal and package frameworks are proposals

The Portal Framework and generic package/AppImage framework are deliberate architecture proposals. They are useful for evaluating future desktop integration, but they are not current stable application APIs.

In particular, current builds should not be assumed to provide a public API for applications to:

- change Meldframe default file handlers;
- set desktop wallpaper through a Meldframe portal;
- register persistent launchers through a stable portal;
- request scoped cross-runtime `FileRef` grants;
- install AppImages through a finished first-party installer;
- use `xdg-desktop-portal-meldframe`.

See [Portal Framework](PORTALS.md) and [Package installation / AppImage](PACKAGE_INSTALLATION.md).

## Browser automation and Electron compatibility remain research/roadmap work

Browser Broker, Playwright/Puppeteer integration with Android Chromium and an Electron-compatible split runtime are not current production features. WebView support and service-backed Web apps should not be generalized into a claim that arbitrary browser automation or Electron applications already work.

## How to interpret a missing feature

Before filing a compatibility bug, distinguish these cases:

| Observation | Likely meaning |
| --- | --- |
| Capability says `AvailableWithSetup` | A known permission/runtime/provider setup step is missing. |
| Capability says `Unknown` | Meldframe has not proved support or failure. This is not the same as unsupported. |
| Capability says `Experimental` | A path exists but is not a stable support promise. |
| Capability says `Broken` | A path expected to work failed its probe or health check. |
| Feature appears only in a proposal/research page | It is not a current product capability. |
| Shell/ADB experiment works but Meldframe cannot do it | The app may lack the authority or provider integration required for the same operation. |

When reporting a problem, include the capability state and diagnostics rather than only the device model. See [Troubleshooting](TROUBLESHOOTING.md).

## Source-of-truth rule

This page is a usability summary, not an implementation contract. If it disagrees with the current implementation repository, executable tests or implementation-side architecture documents, the implementation repository wins and this page should be corrected.
