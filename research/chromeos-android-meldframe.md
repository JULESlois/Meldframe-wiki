# ChromeOS, Android and Meldframe

**Status:** research  
**Last reviewed:** 2026-09-18

## Research question

ChromeOS is the closest mature reference for a desktop that combines Web, Android and Linux
applications. Meldframe should learn from its subsystem boundaries without becoming "ChromeOS on
Android".

The central distinction is:

> ChromeOS primarily normalizes multiple application ecosystems. Meldframe aims to normalize the
> composition of an application across multiple execution and presentation environments.

A logical application may therefore have several valid execution plans while preserving one
`AppId` and one task/window identity.

## Architecture correspondences

| ChromeOS | Meldframe direction | Why it matters |
| --- | --- | --- |
| App Service / publishers | App sources + AppDescriptorRepository | UI surfaces should not understand each ecosystem. |
| App Service instance tracking | WindowRegistry / task model | Application identity and live instances are different facts. |
| Concierge / Cicerone | Runtime manager / broker | Runtime lifecycle and host↔guest coordination are first-class services. |
| Garcon | Guest Agent | Guest-side open-URL, app discovery and host integration belong behind a protocol. |
| Seneschal | FileBridge + FileGrant | Share explicit resources instead of exposing an entire host filesystem. |
| Sommelier / Exo | WaylandPresentationProvider | A guest toplevel can become a host-native window without embedding a whole desktop. |
| System Web Apps | Web/Wasm shell apps | Web UI can host privileged system applications when host APIs are narrow and permissioned. |
| ARCVM / Crostini | AVF/VM RuntimeProvider | Stronger isolation is a runtime property, not an application identity. |
| Lacros / crosapi | logical/deployment separation | API decoupling does not require every component to become a separate process or release train. |

## The App Service lesson

A desktop has many consumers of application data — Start, taskbar, Search, Open With, Settings — and
many publishers — Android, Web, Linux, internal apps and remote apps. Direct pairwise integration
grows roughly as consumers × publishers.

The stable architecture is instead:

```
publishers → common app model → consumers
```

Meldframe already follows this with typed `AppId`, `AppDescriptorRepository`,
`ApplicationCoordinator` and `WindowRegistry`. The next step is not another app type. It is
allowing one logical app to produce several candidate plans.

## Application composition

Example:

```
Code
├─ Linux VS Code          + Wayland
├─ code-server            + Android WebView
├─ Electron Node backend  + Android WebView
└─ remote code-server     + local WebView
```

The planner should decide from capabilities, runtime health, startup cost, performance, isolation and
user policy. Backend choice must not leak into application identity.

This is the strongest conceptual difference from a conventional "Android + Linux subsystem" design.

## Host ↔ guest integration

Crostini's decomposition suggests a dedicated Meldframe Guest Protocol instead of accumulating
Termux-specific intents and scripts.

Initial semantic surface:

```
Hello / protocol negotiation
Health
Exec / process lifecycle
PTY
Service lifecycle
Application discovery
Open URL / Open file
File grants
Clipboard
Notifications
Endpoint publication
Shutdown
```

Potential implementations:

```
External Termux helper
Meldframe Runtime companion
PRoot guest agent
AVF Linux guest
SSH / remote agent
WSL agent
```

The protocol is the stable boundary. The runtime technology is replaceable.

## Runtime and isolation

PRoot is useful for compatibility and deployment convenience, but it is not a strong security
boundary. Runtime descriptions should include isolation/security properties in addition to feature
capabilities.

Candidate runtime family:

```
ExternalTermuxProvider
MeldframeRuntimeProvider
ChrootProvider
AvfLinuxProvider
SshRuntimeProvider
```

AVF matters because modern Android now has an official Linux-development-environment path involving
AVF/crosvm, a Debian-based guest, guest integration and WebView/terminal presentation. Meldframe
should treat AVF as a concrete provider research target rather than a vague future VM option.

## Wayland integration

The first goal is not zero-copy. The first goal is correct independent-window semantics.

Recommended progression:

```
M4a  xdg_toplevel lifecycle + copied/shared-memory frames
M4b  damage-aware copy
M4c  dma-buf import
M4d  dma-buf / AHardwareBuffer / SurfaceControl fast path
```

Meldframe should normally let Android own physical composition and use the bridge to translate
Wayland/application semantics into Android surfaces/windows.

## Logical extension vs deployment unit

Do not equate "plugin" with "separate APK".

Examples:

```
Terminal logical extension
→ built into main APK initially

Wayland presentation extension
→ separate process when licensing/isolation requires it

Formatter extension
→ WASI sandbox

Meldframe Runtime provider
→ separate Android companion APK
```

Choose deployment boundaries from security, licensing, crash isolation, update cadence and
performance after the semantic API is stable.

## Browser and split-runtime implication

ChromeOS already demonstrates guest `xdg-open` forwarding to the host browser. Meldframe can
generalize this:

```
Linux program
    ↓
Browser / Automation shim
    ↓
Meldframe Browser Broker
    ↓
Android WebView / Chrome / dedicated Chromium / remote browser
```

This enables the more aggressive split-runtime model:

```
Linux backend + Android-native presentation
```

Examples include code-server, Jupyter, compatible Electron apps and Playwright workloads using an
Android browser.

## Strategic non-goals

- Do not recreate Ash, SurfaceFlinger or a full compositor unless Android exposes a genuine gap.
- Do not force every Linux application through one runtime.
- Do not make Chromium the shell foundation.
- Do not turn every logical extension into an IPC boundary.
- Do not grant raw Android/system APIs to Web or Wasm extensions.
- Do not let Terminal become synonymous with "the Linux runtime".

## Primary sources

See [the source index](../sources/README.md). This note should be updated when source behavior or
Android platform capabilities materially change.
