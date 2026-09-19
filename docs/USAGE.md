# Using Meldframe

**Audience:** users and testers  
**Status:** user documentation for development builds

Meldframe is evolving quickly. Exact setup can vary by build and device, and a capability shown as
`AvailableWithSetup` may require an external runtime, a permission or a privileged provider.

This guide describes workflows implemented by current development builds. Where a behavior has only
been exercised on WSA, that is stated explicitly rather than treated as general Android validation.

## What you can use today

Current development builds have concrete user-facing paths for:

- Meldframe-managed desktop windows and taskbar interactions;
- the built-in multi-tab Browser;
- Terminal through the currently verified ttyd transport and Termux runtime path;
- Code as a supervised code-server service presented through a Meldframe WebView;
- discovery and Start-menu listing of registered Linux `.desktop` applications.

The last item has an important boundary: **Linux application discovery is implemented, but Linux GUI
launch is not.** Seeing a Linux entry in Start does not imply that selecting it can launch the GUI.
Wayland/XWayland presentation, general Linux GUI execution, AVF runtimes and remote runtimes remain
roadmap or research work unless a more specific page says otherwise.

## Basic desktop use

At the shell level, applications are presented through a common catalogue and launch path. A user
should not need to know whether an entry is backed by an Android Activity, an internal Meldframe
application, a WebView application, or a Linux-backed service. Future remote, Wasm and general Linux
GUI backends are architecture directions, not evidence that those backends are available today.

Backend details belong in diagnostics and advanced settings.

### Taskbar and windows

Current builds implement desktop-style taskbar behavior for Meldframe-managed windows:

- clicking a dormant application button opens it;
- clicking the application that is currently in front minimizes its focused window;
- clicking an application with existing windows while another application is in front raises its
  most-recent window;
- when an application has multiple windows, the taskbar can present an instance switcher;
- **Middle-click** or **Shift+click** opens another window of an application where the application
  supports multiple instances;
- a running application's taskbar menu can offer **Open new window** and **Close window** or
  **Close all windows**. Close commands are only offered when the shell actually owns windows for
  that entry.

These interaction rules are implemented in the shell policy and were exercised on WSA (Android 13)
in September 2026, including open/minimize/restore cycles, opening a second Explorer window, the
multi-instance switcher and closing all Explorer windows. That is useful implementation evidence,
but it is **not** broad OEM/device validation.

Android applications outside Meldframe's own window host are a separate problem. Observing or
controlling Android system tasks may require a privileged provider and must not be inferred from the
behavior above. See [Android desktop integration](ANDROID_DESKTOP.md) and
[Compatibility and verification](COMPATIBILITY.md).

## Built-in Browser

Browser is a Meldframe-owned multi-tab WebView application. Current builds provide tab creation,
switching and closing, back/forward/refresh navigation, address-or-search resolution and a local
Meldframe new-tab page. Entering `javascript:` in the address bar is refused.

The multi-tab flow has been exercised on WSA, including closing the final tab as a Browser-window
close. This is an internal browser application; it is **not** the proposed Browser Broker, a
Playwright/Puppeteer endpoint or a Chromium-extension compatibility layer.

See [Browser](BROWSER.md).

## Capability check first

Before troubleshooting a feature, check its capability state.

A missing feature is not necessarily unsupported. For example:

- `AvailableWithSetup` means a required package/permission/setup step is missing;
- `Broken` means Meldframe expected the feature to work but observed failure;
- `Unknown` means it has not been proven either way.

`Unknown` should not be interpreted as either `Available` or `Unsupported`. This distinction matters
especially for runtime and presentation probing, where an unavailable probe can otherwise produce a
false compatibility claim.

See [Capabilities](CAPABILITIES.md).

## External Termux runtime

The current first-party development path uses external Termux as the initial Linux runtime provider.

For the current `RUN_COMMAND` integration, Termux needs:

1. `allow-external-apps=true` in `~/.termux/termux.properties`;
2. the `com.termux.permission.RUN_COMMAND` permission granted to Meldframe.

Meldframe distinguishes “Termux is installed” from “the runtime is actually ready”. A successful
command round-trip is what upgrades the runtime to ready.

The long-term project direction is to make this easier through a dedicated Meldframe Runtime
companion while retaining external Termux as a supported provider. PRoot/chroot, AVF, SSH and remote
execution should not be inferred from the existence of the runtime abstraction.

## Terminal

The currently verified Terminal path is:

```
Meldframe Terminal
    ↓
xterm.js / WebView
    ↓
TerminalTransport
    ↓
ttyd
    ↓
selected runtime / shell
```

A profile chooses a **runtime target**, not just a shell command. The abstraction is intended to allow
other transports and runtimes later, but direct Termux PTY, Runtime Agent, local PTY and SSH
transports are not current production transports merely because the interface can represent them.

Recorded WSA testing covers real shell I/O, resize propagation, title updates and common keyboard
sequences through ttyd. Rich tab-strip UI, multiple visible panes, reconnect semantics and a direct
Meldframe PTY transport remain ongoing work. A test suite that skips real ttyd integration because
the binary is absent is not equivalent to an end-to-end transport pass.

See [Terminal](TERMINAL.md).

## Code / code-server

Code demonstrates service-backed applications. Its current launch path is approximately:

```
Termux/runtime ready
 ↓
start code-server
 ↓
wait for /healthz
 ↓
Meldframe loopback gateway/session
 ↓
Android WebView
 ↓
workbench + WebSocket/extension host
```

The user interacts with one Meldframe window even though the backend runs in the Linux runtime. The
service must become healthy before the application window is considered successfully launched;
otherwise Meldframe reports the service state rather than treating a blank WebView as success.

The token-gated loopback gateway reduces exposure but is not a complete sandbox: the backend's own
loopback listener still exists. Moving the backend side to a Unix socket is a further hardening
direction, not current behavior.

See [Code / service-backed apps](CODE.md).

## Linux applications in Start

Current builds can register an Android-readable directory containing Linux `.desktop` entries, parse
eligible entries and merge them into the common catalogue. Identity is scoped by container/runtime
registration so two guests do not have to share one global desktop ID. The parser also applies
relevant desktop-entry visibility/localization rules and converts `Exec` into argv rather than
executing it as a shell script.

This path has been checked against real WPS Office and Cylheim desktop entries. Registration is still
a development/test workflow exposed through the opt-in ADB acceptance bridge, not a finished
end-user package installer.

If a registered Linux application appears in Start but does not launch, do **not** troubleshoot it as
if a Wayland session had failed. The current product has no production `AppId.Linux` GUI launch
backend. The Linux GUI whole-display work is research evidence and does not change that boundary.

See [Linux application discovery](LINUX_APPS.md) and [Troubleshooting](TROUBLESHOOTING.md).

## ADB acceptance bridge for testers

Development builds expose an ADB-oriented acceptance bridge for exercising catalogue, launch and
window-command paths without coordinate tapping. It is disabled by default. Enable it under
**Settings > Privacy and security > For developers > Shell bridge** when performing acceptance work,
and disable it again afterward.

The Settings switch is not the authorization boundary by itself. The exported receiver additionally
requires Android's privileged/signature `DUMP` permission, so an ordinary installed application
cannot invoke it merely because the switch is on.

The accompanying `mf.sh` commands and Linux registration operations are development/acceptance
interfaces. They are **not** a promised stable public CLI or automation API.

See [Testing Meldframe with ADB](TESTING_AND_ADB.md) and [Installation and setup](INSTALLATION.md).

## Diagnostics

When behavior differs across Android devices, collect Meldframe diagnostics before assuming the
device is unsupported.

Diagnostics are designed to report:

- desktop/session capability state;
- runtime providers and health;
- failed probes and their reasons;
- presentation/backend availability;
- extension state;
- relevant GPU/system integration information.

The project deliberately avoids including installed application names in the general diagnostics
report; counts are sufficient for catalogue-health checks.

For Linux/runtime failures, separate the layers before drawing a conclusion: application discovery,
execution/runtime readiness and GUI presentation are distinct stages. A success at one stage is not
evidence that the next stage exists.

## Advanced and privileged features

Some desktop/task/system operations require more authority than a normal Android application has.
Depending on the feature and device, Meldframe may use or investigate:

- normal Android APIs;
- explicit user-granted setup;
- Shizuku/shell authority;
- root helper;
- privileged/system application integration;
- custom-ROM/OS-tier integration.

These mechanisms are not interchangeable, and their presence in the architecture does not establish
that a particular operation is supported on a particular device. A normal APK should remain useful
even when privileged integrations are unavailable.

## Where to go next

- [Features and status](FEATURES.md)
- [Capabilities](CAPABILITIES.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Compatibility and verification](COMPATIBILITY.md)
- [Known limitations](KNOWN_LIMITATIONS.md)
- [Architecture overview](ARCHITECTURE.md)
