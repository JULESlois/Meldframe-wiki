# Installation and setup

**Audience:** users and testers  
**Status:** development-build documentation

Meldframe is not yet documented as a stable end-user release. This page therefore describes the
current setup model without pretending that there is a finished installer or a fixed compatibility
matrix.

## Before installing

Meldframe is an Android application, but some of its more interesting workloads depend on optional
providers outside the APK. Installing Meldframe alone and installing a Linux runtime are separate
operations.

A useful mental model is:

```
Meldframe APK
├─ Android applications and shell features
├─ built-in Terminal / Code / Wasm extensions
└─ optional runtime/system providers
   ├─ external Termux             current
   ├─ privileged Android access   capability-dependent
   └─ AVF / chroot / SSH          planned or research
```

The shell should remain useful when an optional provider is absent. Missing setup should normally be
reported as `AvailableWithSetup` or an equivalent runtime state rather than crashing the desktop.

## Current Linux runtime path: external Termux

The first real `RuntimeProvider` uses Termux's `RUN_COMMAND` service. The implementation repository
has verified this path on WSA; that does **not** imply that every Android/OEM combination has been
verified.

For the current integration, Termux must allow external command requests:

```properties
# ~/.termux/termux.properties
allow-external-apps=true
```

Meldframe also needs Termux's `com.termux.permission.RUN_COMMAND` permission. Merely detecting the
Termux package is not enough: the provider runs a real `uname -a` command and only reports the
runtime ready after the command returns successfully.

If Termux is installed but command execution fails, treat that as a setup/runtime problem rather than
proof that Linux execution is unsupported on the device.

## Terminal dependencies

The current Terminal PoC uses ttyd as one transport:

```
shell ↔ PTY ↔ ttyd ↔ WebSocket ↔ Meldframe Terminal ↔ xterm.js
```

ttyd is not the Terminal architecture and is not intended to be the only transport. It is a
bootstrap transport used to validate PTY, byte I/O, resize, title updates and desktop interaction.

The current implementation has been verified against ttyd 1.7.7. Other versions should not be
assumed incompatible, but they are not covered by that verification result.

## Code dependencies

The Code extension uses code-server as a supervised Linux service and presents it in a
Meldframe-owned WebView.

The implementation has verified Termux User Repository `code-server 4.137.0` on WSA. The version is
a recorded test point, not a permanent Meldframe requirement.

Meldframe waits for the service health endpoint before treating the application as ready. The
WebView reaches the service through Meldframe's loopback token proxy rather than exposing an
unauthenticated service directly as the application boundary.

## Privileged desktop features

Some task/window/system operations cannot be implemented by an ordinary Android APK on every
platform. Meldframe's capability model is designed to distinguish normal Android access from
possible shell/Shizuku, root, privileged/system or OS-tier providers.

Do not grant root or other privileged access merely because Meldframe can make use of it. Enable a
privileged provider only for a feature you understand and need. PRoot, in particular, is a
compatibility mechanism and should not be treated as a security sandbox.

## What is not an installation requirement today

The following are roadmap/research directions, not prerequisites for the current development build:

- Android Virtualization Framework (AVF) Linux guests;
- a dedicated Meldframe Runtime companion APK;
- wlroots/labwc or a production Wayland bridge;
- a third-party Meldframe plugin package manager;
- a dedicated Chromium/Playwright broker;
- a custom ROM or SystemUI replacement.

If a guide claims one of these is mandatory for current Meldframe, it is ahead of the implementation.

## After setup

Use the capability/runtime diagnostics to verify the environment rather than inferring readiness
from installed packages. Then continue with:

- [Using Meldframe](USAGE.md)
- [Terminal guide](TERMINAL.md)
- [Code guide](CODE.md)
- [Capabilities](CAPABILITIES.md)
- [Troubleshooting](TROUBLESHOOTING.md)
