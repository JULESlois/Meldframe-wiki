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

The first real `RuntimeProvider` uses Termux's public `RUN_COMMAND` service. The implementation
repository has verified this path on WSA; that does **not** imply that every Android/OEM combination
has been verified.

There are two independent setup requirements.

### 1. Allow external commands in Termux

In Termux, add this line to `~/.termux/termux.properties`:

```properties
allow-external-apps=true
```

Then reload Termux settings:

```sh
termux-reload-settings
```

Restarting Termux is a reasonable fallback if the setting does not appear to take effect.

### 2. Grant Meldframe Termux's RUN_COMMAND permission

Meldframe must hold:

```text
com.termux.permission.RUN_COMMAND
```

This permission is not evidence that the runtime works by itself. In the WSA development setup used
by the implementation repository, the permission was granted explicitly with ADB:

```sh
adb shell pm grant top.cmys.meldframe com.termux.permission.RUN_COMMAND
```

That command is a **recorded development/test setup path**, not a claim that ADB should be a permanent
end-user installation requirement. If packaging or an in-app permission flow changes in the
implementation repository, follow the implementation rather than preserving this command as lore.

### 3. Verify the probe, not just the packages

After both requirements are satisfied, Meldframe's provider runs a real `uname -a` command through
Termux and waits for its result. A healthy runtime is therefore stronger evidence than “Termux is
installed”.

The current diagnostics are intentionally specific about missing setup:

- missing `RUN_COMMAND` permission → runtime needs setup;
- Termux refusing external execution → check `allow-external-apps=true`;
- command/result failure → preserve the reported reason rather than treating package presence as
  success.

On the recorded WSA verification, both setup steps produced a `READY` Termux runtime and a successful
`uname -a` result. This is a test point, not an Android-wide compatibility guarantee.

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

## Optional ADB shell bridge for acceptance testing

Meldframe contains an ADB-facing shell bridge so testers can drive the **real composed shell** without
fragile coordinate taps. It can list applications/windows and route launch/window commands through
the same launcher and `WindowCommands` objects used by the UI. This is a test/control surface, not a
normal application API and not a prerequisite for using Meldframe.

The bridge now exists in release builds, but a fresh install keeps its Android receiver component
disabled. To opt in, open:

**Settings → Privacy and security → For developers → Shell bridge**

Turning the switch on is only the first of two gates. The receiver also requires Android's
`android.permission.DUMP`, a `signature|privileged` permission available to ADB shell/the system, not
to an ordinary installed application. The switch therefore expresses user intent; it is **not** the
security boundary that makes the receiver safe to export.

Turning the switch off disables the receiver component itself rather than merely making an enabled
receiver reject commands. On startup Meldframe reapplies the stored preference to the component, so
an external `pm enable` does not permanently make the UI report a contradictory state.

The implementation repository verified this behavior on WSA: a fresh install left the receiver
disabled and shell commands silent; enabling the developer switch made the bridge answer; disabling
it made the bridge silent again. Do not infer from that acceptance result that ADB automation is an
end-user feature or that non-ADB apps can call the bridge.

The repository's `mf.sh` helper is the current acceptance client. Commands documented in implementation
commits include `apps`, `launch`, `windows`, `minimize`, and the Linux registration helpers used by
current development. Treat that script as a development interface: command names and output are not
a stable public API unless the implementation explicitly promotes them to one.

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
