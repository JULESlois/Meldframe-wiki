# Using Meldframe

**Audience:** users and testers  
**Status:** user documentation for development builds

Meldframe is evolving quickly. Exact setup can vary by build and device, and a capability shown as
`AvailableWithSetup` may require an external runtime, a permission or a privileged provider.

This guide describes the intended workflow and the runtime setup that is already exercised by the
project.

## Basic desktop use

At the shell level, applications are presented through a common catalogue and launch path. A user
should not need to know whether an entry is backed by:

- an Android Activity;
- an internal Meldframe application;
- a WebView application;
- a Linux service;
- a Linux GUI program;
- a future remote or Wasm backend.

Backend details belong in diagnostics and advanced settings.

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
frontend.

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
Depending on the feature and device, Meldframe may eventually use:

- normal Android APIs;
- explicit user-granted setup;
- Shizuku/shell authority;
- root helper;
- privileged/system application integration;
- custom-ROM/OS-tier integration.

These tiers are additive. A normal APK should remain useful even when privileged integrations are not
available.

## Where to go next

- [Features and status](FEATURES.md)
- [Capabilities](CAPABILITIES.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
