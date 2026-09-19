# Using Meldframe

**Audience:** users and testers  
**Status:** user documentation for development builds

Meldframe is evolving quickly. Exact setup can vary by build and device, and a capability shown as
`AvailableWithSetup` may require an external runtime, a permission or a privileged provider.

This guide describes workflows implemented by current development builds. Where a behavior has only
been exercised on WSA, that is stated explicitly rather than treated as general Android validation.

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

## Capability check first

Before troubleshooting a feature, check its capability state.

A missing feature is not necessarily unsupported. For example:

- `AvailableWithSetup` means a required package/permission/setup step is missing;
- `Broken` means Meldframe expected the feature to work but observed failure;
- `Unknown` means it has not been proven either way.

See [Capabilities](CAPABILITIES.md).

## External Termux runtime

The current first-party development path uses external Termux as the initial Linux runtime provider.

For the current `RUN_COMMAND` integration, Termux needs:

1. `allow-external-apps=true` in `~/.termux/termux.properties`;
2. the `com.termux.permission.RUN_COMMAND` permission granted to Meldframe.

Meldframe distinguishes “Termux is installed” from “the runtime is actually ready”. A successful
command round-trip is what upgrades the runtime to ready.

The long-term project direction is to make this easier through a dedicated Meldframe Runtime
companion while retaining external Termux as a supported provider.

## Terminal

The current terminal path is conceptually:

```
Meldframe Terminal
    ↓
xterm.js / WebView
    ↓
TerminalTransport
    ↓
ttyd or another transport
    ↓
selected RuntimeInstance / shell
```

A profile chooses a **runtime target**, not just a shell command. This is why the same Terminal UI can
eventually connect to Termux, PRoot/chroot, AVF, SSH or a remote agent without changing the terminal
frontend. “Eventually” is important here: those targets do not all have production providers today.

Current builds should be treated as development/PoC quality for advanced interaction such as
multi-pane UI, reconnect, physical-keyboard edge cases and IME behavior.

## Code / code-server

Code demonstrates service-backed applications.

The intended launch sequence is:

```
Code
 ↓
RuntimeProvider
 ↓
ServiceSupervisor
 ↓
code-server
 ↓
Meldframe service gateway
 ↓
Android WebView
```

The user interacts with one Meldframe window even though the backend runs in the Linux runtime.

The service must become healthy before the application window is considered successfully launched;
otherwise Meldframe reports the service state rather than opening a blank WebView.

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
