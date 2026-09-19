# Runtimes

**Audience:** users, extension authors and contributors  
**Status:** explanatory documentation; implementation status checked against `Hyperdroid-recovery` main on 2026-09-19

A **runtime** is where non-Android work executes. It is not the same thing as an application, a terminal, a Linux distribution, or a presentation backend.

## The three concepts that are easiest to confuse

| Concept | Example | Meaning |
| --- | --- | --- |
| Runtime provider | Termux | integration that discovers/exposes execution environments |
| Execution environment/backend | native Termux, future PRoot/chroot/VM | mechanism used to execute work |
| Runtime instance | the currently exposed `termux` instance | a concrete environment the planner can target |

Application identity should not depend on which provider happens to host it. This is why migrating a future Linux application between PRoot, chroot or another provider should not require it to become a different Start-menu application.

## What exists today

The current implementation has a real `RuntimeProvider` abstraction and runtime catalogue. It also has a registered Termux integration that can verify external command execution by sending `uname -a` through Termux `RUN_COMMAND`.

The Termux instance reports one of these states:

- `UNKNOWN` — command execution has not been verified yet;
- `READY` — the probe completed successfully;
- `NEEDS_SETUP` — Termux exists but Meldframe cannot currently execute through it;
- `UNAVAILABLE` — Termux is not installed.

This distinction matters. **Termux being installed is not evidence that the runtime is usable.** Meldframe also needs `RUN_COMMAND` permission and Termux must accept commands from external applications.

The current Termux runtime advertises **WebView presentation only**. That means a process can back a loopback Web service such as ttyd or code-server and Meldframe can present it through Android WebView. It does not mean arbitrary Linux GUI applications can currently appear as independent Meldframe windows.

## Guest capability probing

The implementation now contains a structured `GuestCapabilityProbe`. It asks an execution environment about concrete properties instead of inferring them from a provider name. Current probe fields cover:

- glibc compatibility;
- PRoot/proot-distro presence;
- a live Wayland socket;
- an X11 socket;
- XWayland availability;
- DRM render nodes;
- a Vulkan loader.

Each answer is tri-state: `PRESENT`, `ABSENT`, or `UNKNOWN`. `UNKNOWN` deliberately means that the question could not be answered; it is not silently converted to `ABSENT`. The probe also retains short evidence from the command result so a capability decision can be traced back to what the guest actually reported.

Presentation capability is derived conservatively. A Wayland backend is only advertised from a positive Wayland probe. XWayland requires both XWayland and Wayland to be positively present. An unknown probe therefore cannot make the planner select a presentation path that merely *might* work.

This is **implemented capability-model/probe logic**, not evidence that every runtime provider already runs these probes or that a production Wayland backend exists. In particular, the current user-facing Termux path remains WebView-based.

## Setup state versus runtime state

A common setup failure is:

```text
Termux installed
      ↓
RUN_COMMAND permission missing
      ↓
NEEDS_SETUP
```

Another is:

```text
permission granted
      ↓
Termux refuses external command
      ↓
set allow-external-apps=true
      ↓
verify again
```

See [Installation](INSTALLATION.md) and [Troubleshooting](TROUBLESHOOTING.md) for the user-facing setup path.

## What Terminal adds

Terminal is not itself a runtime provider. It consumes runtime/transport facilities and presents an interactive terminal UI.

The current verified Terminal path uses ttyd as a loopback WebSocket/Web terminal bridge. That is a useful proof of the runtime boundary, but it is not yet a general PTY protocol with durable reattachment semantics.

See [Terminal](TERMINAL.md).

## What Code adds

Code is another consumer of runtime infrastructure. Its backend runs as a supervised service and its UI is presented through Android WebView.

Conceptually:

```text
Termux / Linux-side service
        ↓
code-server
        ↓
localhost gateway
        ↓
Android WebView
        ↓
one Meldframe application/window identity
```

This validates split execution/presentation without claiming that WebView is the intended presentation path for every Linux GUI application.

See [Code](CODE.md).

## Runtime discovery is evidence, not a guess

The runtime model deliberately avoids rules such as:

```text
Termux package installed → Linux ready
Linux app → Wayland available
rooted device → chroot works
```

Those implications are not reliable.

The current guest probe establishes an initial evidence surface for libc/userspace compatibility, PRoot, Wayland, X11/XWayland and graphics prerequisites. Other runtime properties still need equivalent evidence where they become planner inputs, including long-running-service behavior, Unix sockets, localhost TCP, shared storage, audio and architecture compatibility.

Only capabilities that have actually been detected should influence the planner as facts.

## Linux GUI research: what has actually been proven

A 2026-09-19 M3 development experiment exercised a whole-display Linux GUI chain using WSL as the runtime because the WSA Termux guest used for development had no network access for installing the required stack. The test chain was:

```text
WSL
  → nested sway on WSLg
  → Cylheim 4.11.1
  → wayvnc / RFB
  → websockify / noVNC
  → adb reverse
  → Meldframe WebView
```

The experiment proved, separately, that a real Linux GUI application ran, sway rendered it, and wayvnc delivered a complete rendered frame over the RFB socket. Meldframe could load the noVNC client and establish the WebSocket session.

The final browser presentation did **not** succeed: noVNC remained blank. The same failure reproduced in headless Edge outside Meldframe, narrowing the remaining problem to the noVNC/neatvnc development harness rather than establishing a Meldframe WebView defect.

This result is useful engineering evidence, but it is not a shipped feature. It does **not** mean Meldframe currently has Linux GUI launch support, a Wayland window backend, VNC desktop support, or a supported WSL runtime provider. The experiment validates part of the proposed presentation chain and records where the PoC stops.

## Current support status

| Runtime path | Current status | What that means |
| --- | --- | --- |
| External Termux native execution | **Implemented and device-verified on WSA** | `RUN_COMMAND` probe can establish a ready runtime and service-backed apps have been exercised |
| ttyd loopback runtime/transport | **Implemented and device-verified on WSA** | useful for Terminal; not equivalent to a native PTY reattach protocol |
| code-server service path | **Implemented and device-verified on WSA** | service-backed WebView application path works |
| Structured guest capability probe | **Implemented core probe logic** | can interpret glibc/PRoot/Wayland/X11/XWayland/DRM/Vulkan evidence; do not assume every provider is wired to it |
| Whole-display Linux GUI chain | **Research PoC, partial success** | application → compositor → RFB frame was proven; noVNC presentation remained broken in the development harness |
| PRoot/proot-distro as a general discovered runtime | **Architecture/roadmap, not a general current provider** | do not assume installed distros are automatically discovered or selectable |
| root/chroot | **Candidate/roadmap** | no general production provider should be inferred from root availability |
| AVF Linux VM | **Research/roadmap** | provider design is discussed, not a current user feature |
| SSH/remote runtime | **Candidate/roadmap** | architecture permits it; no stable user-facing provider is documented as current |
| Wayland GUI runtime presentation | **Roadmap** | probe/model work and a whole-display PoC exist, but the current Termux provider still advertises WebView rather than a production Wayland backend |
| XWayland | **Roadmap** | compatibility presentation, not current verified product support |

The table intentionally distinguishes architecture compatibility, research evidence and an implementation that users can rely on.

## Provider API: current versus intended

The runtime abstraction has evolved incrementally. Core discovery (`RuntimeProvider.instances()`) is deliberately small; concrete Android integrations add the execution behavior needed to validate real workloads. Service supervision and Termux command execution now exist elsewhere in the implementation.

Do not read older design prose saying “no provider is registered yet” as the current product state. That was true at an earlier milestone and is superseded by the registered Termux integration and WSA verification evidence.

The long-term provider contract is expected to cover:

```text
discover instances
expose capabilities
execute commands
start/stop services
report health
```

The exact stable external plugin API is not frozen yet.

## Why Meldframe does not choose a runtime by app type alone

A future application may have several valid execution plans:

```text
native arm64 Linux runtime
translated x86_64 runtime
remote runtime
service-backed Web runtime
```

The planner should select among candidates using requirements and observed capabilities, not a hard-coded statement such as “all Linux apps use Debian”.

This becomes especially important for package installation: an AppImage can require a particular architecture and Linux userspace. See [Package installation / AppImage](PACKAGE_INSTALLATION.md).

## Related documents

- [Capabilities](CAPABILITIES.md)
- [Linux application discovery](LINUX_APPS.md)
- [Terminal](TERMINAL.md)
- [Code](CODE.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
- [Troubleshooting](TROUBLESHOOTING.md)
