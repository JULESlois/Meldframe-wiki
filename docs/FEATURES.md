# Features and current status

**Audience:** users and evaluators  
**Status:** user documentation

Meldframe is under active development. This page distinguishes **implemented/verified architecture**
from **planned or experimental directions** so the project is not described as more complete than it
is.

## Desktop shell

The shell provides the familiar desktop concepts Meldframe grew from: Start/application surfaces,
taskbar integration, internal windows and a desktop-oriented interaction model.

The newer architecture routes application enumeration and launch through common services rather than
letting every UI surface talk directly to Android package APIs.

Current architecture includes:

- a unified `AppDescriptorRepository`;
- typed application IDs, including Android work-profile and Linux container identity;
- a central `ApplicationCoordinator` and execution planner;
- a backend-neutral `WindowRegistry`;
- capability-aware backend admission;
- desktop session modes and device profiles.

Deeper Android task control depends on the authority available on a device. Ordinary APK, root,
Shizuku/system and future WM Shell integrations are treated as different capability providers.

A separate shell-control bridge now exists for acceptance/testing from `adb shell`. It ships disabled
and is explicitly enabled under **Settings > Privacy and security > For developers**. Enabling the
switch is not itself the security boundary: the exported receiver also requires Android's
signature/privileged `DUMP` permission, so an ordinary installed application cannot invoke it. The
current WSA acceptance run verified disabled-by-default, enable, command response and disable-again
behavior. This bridge should be treated as a developer/test surface, not as a public automation SDK.

## Browser

The internal Browser is now a multi-tab WebView application rather than the earlier single-page
recovery slice.

Current implementation includes:

- one persistent WebView per open tab;
- a tab strip integrated with the window title bar;
- deterministic new-tab, switching and close behavior;
- back, forward and refresh navigation;
- address/search resolution with `javascript:` refused at the address bar;
- a Meldframe-owned local new-tab page;
- standard Meldframe window controls.

The recorded WSA acceptance run verified opening, switching and closing multiple tabs, including
closing the final tab as a Browser-window close. This does **not** mean the proposed Browser Broker,
Playwright/Puppeteer automation or a Chromium extension platform is implemented.

See [Browser](BROWSER.md).

## Meldframe Terminal

Terminal is the first runtime application used to validate the Linux/runtime architecture.

Implemented pieces include:

- xterm.js presentation inside a Meldframe-owned WebView;
- byte-oriented terminal protocol;
- terminal profiles;
- sessions, tabs and pane-domain models;
- pluggable `TerminalTransport`;
- ttyd transport;
- a minimal RFC 6455 WebSocket client for the local ttyd path;
- runtime selection as part of a terminal profile.

The first real-device/WSA PoC validated shell I/O, resizing, title updates and common keyboard
sequences. Rich tab-strip UI, multiple visible panes, reconnect semantics and a direct Meldframe PTY
transport remain ongoing work.

## Linux runtime integration

The first real RuntimeProvider uses external Termux.

The current path can:

- detect the Termux host and whether setup is complete;
- invoke commands through Termux `RUN_COMMAND`;
- collect output and exit status;
- start service-backed workloads;
- report health into the runtime/capability model.

Meldframe can also now register a guest/container `.desktop` directory, parse its application entries
and merge them into the common application catalogue with container-scoped Linux identities. The
parser handles desktop-entry visibility/localization and resolves `Exec` to argv without handing it to
a shell. This discovery path has been checked against real WPS Office and Cylheim entries.

**Linux application discovery is not Linux GUI execution.** Discovered entries may appear in Start,
but the current build deliberately refuses to launch `AppId.Linux` because the GUI execution and
presentation chain does not exist yet. Registration is currently exposed through the opt-in ADB test
bridge rather than a finished end-user installer. See [Linux application discovery](LINUX_APPS.md).

Longer-term providers are expected to include a curated Meldframe Runtime companion, chroot,
Android Virtualization Framework (AVF) guests, SSH and remote runtimes.

## Code / code-server

Code is the first service-backed compatibility application.

The current architecture treats code-server as a Linux service while presenting it as a Meldframe
application through Android WebView. The shell supervises the service, checks health before opening
the window and places a token-gated loopback gateway between the WebView and the service.

This validates an important Meldframe pattern:

```
Linux/service backend
        +
Android-native presentation
        =
one Meldframe application
```

## Web and Wasm

The extension system already has capability probes, presentation contributions and a Wasm
application path.

A current WebView-based Wasm probe executes real modules instead of assuming support from a browser
version. WebAssembly and SIMD have been validated on the existing WSA test environment; threads,
WASI and GPU-related features remain capability-dependent.

Meldframe intentionally treats Wasm as a runtime/capability technology rather than assuming it must
replace Android View or DOM rendering.

## Extensions

The current extension registry supports typed contribution points rather than a single catch-all
Plugin interface. Existing built-in extensions already prove several different shapes:

- Terminal: App + Runtime + Terminal contributions;
- Code: ServiceApp contribution;
- Wasm: CapabilityProbe + PresentationBackend + WasmApp contributions.

External third-party extension packaging is **not yet a stable public format**. The current model is
being validated with built-in extensions before the process/package boundary is frozen.

See [Extensions and plugins](EXTENSIONS.md).

## Capability-aware behavior

Meldframe does not treat support as a boolean. Features can be:

`Available`, `AvailableWithSetup`, `Experimental`, `Broken`, `Unsupported`, or `Unknown`.

This is used for device-specific desktop behavior, runtime availability, Web/Wasm features,
presentation backends and system integration.

See [Capabilities](CAPABILITIES.md).

## Active development areas

Important directions that should currently be read as roadmap work, not completed product promises:

- Linux GUI execution for the newly discoverable `.desktop` applications;
- per-window Linux GUI integration through Wayland;
- XWayland compatibility;
- dma-buf/AHardwareBuffer/SurfaceControl fast paths;
- AVF-backed Linux runtime;
- Meldframe Runtime companion;
- Browser Broker and Playwright/Puppeteer integration with Android Chromium;
- Electron-compatible split runtime;
- unified FileRef/FileBridge across Android/Linux/remote files;
- Inspector/debug adapters;
- external extension package loading and sandboxing;
- deeper SystemUI/Quickstep/WM Shell integration;
- multiple shell personalities.

For the architecture behind those directions, see [Architecture overview](ARCHITECTURE.md).
