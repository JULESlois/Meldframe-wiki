# Architecture overview

**Audience:** technical users, extension authors and contributors  
**Status:** explanatory documentation

Meldframe is easiest to understand as a composition layer above Android.

```
                         AppDescriptor
                              │
                       Application Core
                              │
                   ┌──────────┴──────────┐
                   │                     │
            Capability Resolver      User Policy
                   │                     │
                   └──────────┬──────────┘
                              │
                      Execution Planner
                              │
       ┌──────────────────────┼─────────────────────┐
       │                      │                     │
 Execution Provider    Presentation Provider   Compatibility Provider
       │                      │                     │
 Android                 Android task             native
 Linux                   WebView                  code-server
 Wasm                    Wayland                  ElectronCompat
 Remote                  remote                   browser shim
       │                      │                     │
       └──────────────────────┼─────────────────────┘
                              │
                        WindowRegistry
                              │
                       Android WM / Shell
                              │
                        SurfaceFlinger
```

## The main separations

### Application identity != runtime

A Linux application remains the same logical application whether it currently runs under external
Termux, a PRoot distro, chroot, AVF VM or a remote host.

### Execution != presentation

A backend may execute in Linux while presenting through Android WebView.

That is how code-server can behave like a normal desktop application without forcing Chromium to run
inside PRoot.

### Process != window

Starting a process does not prove that a window exists. Likewise, a WebSocket connection is not a
terminal process identity.

Meldframe consumes lifecycle facts from whichever subsystem owns them.

### Capability != permission

The system may be technically capable of reading the clipboard while a third-party extension is not
allowed to do so.

### Extension != deployment unit

An extension defines a semantic contribution. It may later be deployed in-process, out-of-process,
as a companion APK or in a sandbox.

## Core objects

Important architecture concepts include:

- `AppId` / `AppDescriptor` — application identity and metadata;
- `ApplicationCoordinator` — common launch entry point;
- `ExecutionPlan` — selected way to execute and present an application;
- `WindowRegistry` — backend-neutral window/task facts;
- `CapabilityGraph` — detected/effective system abilities;
- `RuntimeProvider` / `RuntimeInstance` — where Linux or remote work can execute;
- `ExtensionRegistry` — typed contributions;
- `PermissionBroker` — per-extension authority;
- `ServiceSupervisor` — lifecycle/health for service-backed applications.
- `PortalBroker` — **proposed** semantic desktop-service boundary for applications.

## Runtime family

Meldframe is intentionally not tied to one Linux technology.

The current and planned provider family includes:

```
External Termux
Meldframe Runtime companion
PRoot / distro environments
root/chroot
AVF Linux VM
SSH / remote runtime
```

The host/guest integration is expected to converge on a versioned Meldframe Guest Protocol instead
of every provider inventing unrelated shell scripts and intents.

## Presentation family

Top-level presentation can include:

```
Android task
Meldframe-owned view
WebView
Wayland
XWayland
remote stream
```

WebView can itself expose DOM, Canvas, WebGL, WebGPU or WebAssembly features. Those are better
expressed as capabilities than as a combinatorial explosion of presentation backend IDs.

## Android's role

Meldframe should use Android's strongest existing mechanism whenever possible:

- WM/WM Shell for task/window mechanisms;
- SurfaceFlinger/SurfaceControl for composition;
- Android input and IME;
- WebView/Chromium for web presentation;
- Android media/GPU stacks;
- AVF/crosvm where available for virtualized Linux.

Meldframe's job is primarily policy, orchestration, compatibility and semantic composition.

## Why ChromeOS is an important reference

ChromeOS has already demonstrated several analogous boundaries:

- App Service for heterogeneous application publishers/consumers;
- Crostini runtime lifecycle decomposition;
- Garcon-style guest-to-host integration;
- scoped host file sharing;
- Sommelier/Exo per-window Linux GUI integration;
- privileged System Web Apps.

Meldframe's research direction goes one step further by allowing a single logical application to be
split across execution and presentation environments.

For the detailed comparison, see
[ChromeOS, Android and Meldframe](../research/chromeos-android-meldframe.md).


## Proposed desktop-service layer

Execution and presentation are not enough to make an application feel native to a desktop.
Applications also need desktop services: file choosers, Open With, default handlers, notifications,
wallpaper, shortcuts, launchers, clipboard and other host integrations.

The proposed Portal Framework adds that layer without exposing raw Android internals:

```
Application
    ↓
Portal API
    ↓
PortalBroker
    ├ PermissionBroker
    ├ CapabilityGraph
    └ GrantStore
    ↓
Android / Shell / Runtime provider
```

The same semantic portal can be reached from Kotlin, Web/JavaScript, Wasm/WASI or a Linux guest.
Linux compatibility can later map standard `xdg-desktop-portal` interfaces into Meldframe portals,
while Wayland remains responsible for window/presentation integration.

AppImage Installer is the proposed first package workload for this layer because it exercises
FileRef, runtime selection, architecture compatibility, desktop-entry parsing, MIME associations,
launcher integration and normal application launch without requiring AppImage-specific shell logic.

See [Portal Framework](PORTALS.md), [Package installation / AppImage](PACKAGE_INSTALLATION.md) and the
[research note](../research/portal-appimage-xdg.md).
