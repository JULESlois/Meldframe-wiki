# Windows runtime landscape for Meldframe

**Status:** research  
**Last reviewed:** 2026-09-25

## Research question

What Windows execution backends are useful to Meldframe, and where should their boundaries sit so
that Windows applications can become first-class Meldframe applications without making Winlator,
Wine, Box64, FEX or a particular VM implementation part of Meldframe Core?

The main conclusion is:

> Meldframe should model Windows support as a Windows compatibility/provider family, not as a
> Winlator feature.

Winlator is currently the strongest bootstrap candidate because it already integrates most of the
difficult Android-side compatibility stack. It should not become the permanent architectural
boundary. The long-term provider contract should be able to host a Winlator-derived engine,
ARM64EC/FEX or Hangover-style compatibility engines, and a full Windows ARM VM without changing
application identity or shell semantics.

## Two fundamentally different Windows paths

"Run Windows applications" hides two different systems.

### Compatibility execution

Typical stack:

~~~
Windows executable
        ↓
Wine / Proton
        ↓
Box64 / FEX / native ARM64EC pieces
        ↓
Mesa / Turnip
        ↓
DXVK / VKD3D / WineD3D
        ↓
Android
~~~

This is the Winlator, GameNative and Hangover family.

It is the preferred Meldframe path for ordinary Windows applications because it can avoid booting a
complete Windows operating system and can eventually expose individual application windows to
Meldframe.

### Full Windows virtual machine

Typical stack:

~~~
Windows executable
        ↓
Windows 11 ARM
        ↓
Windows x86/x64 emulation when needed
        ↓
virtual GPU / virtual devices
        ↓
crosvm / QEMU
        ↓
Android virtualization backend
~~~

DroidVM is representative of this family.

This is useful when an application needs behavior that Wine cannot provide, but the natural
presentation unit is initially a whole VM display rather than one host-native application window.

These paths should therefore share a shell-facing Windows application model while remaining distinct
provider implementations.

## Recommended provider model

Meldframe Core should see a semantic compatibility provider:

~~~
Meldframe
    ↓
WindowsCompatibilityProvider
    ↓
CompatibilityEnvironmentRef
    ↓
engine-specific adapter
~~~

Candidate engines may include:

~~~
Winlator-derived compatibility engine
ARM64EC + FEX compatibility engine
Hangover-style engine
Windows ARM VM provider
legacy QEMU provider
~~~

Core should not require knowledge of:

~~~
Winlator Container IDs
Wine prefix paths
Box64 presets
FEX configuration files
DXVK environment variables
Turnip package layout
Winlator shortcut storage
DroidVM disk-image layout
QEMU machine arguments
~~~

Those are provider implementation details.

## Winlator as the first bootstrap engine

Winlator remains the strongest first engineering target because it already composes a large set of
pieces that Meldframe should not recreate merely to prove Windows application support:

- Wine;
- Box64/Box86-oriented CPU translation paths;
- root filesystem/bootstrap logic;
- Wine prefix lifecycle;
- Mesa/Turnip/Zink/VirGL-related graphics integration;
- DXVK/VKD3D/WineD3D;
- X server/display integration;
- audio;
- input and gamepad handling;
- per-application compatibility settings.

Its current application source also exposes useful decomposition points. The display session owns an
X environment assembled from components such as the X server, guest program launcher and audio
services. The internal X server has an explicit window manager and observable map/unmap/window state.

This suggests that the first Meldframe experiment does not need to redesign Winlator graphics.

A useful initial path is:

~~~
Meldframe
    ↓ Binder / local protocol
WinCompat companion
    ↓
Winlator-derived session engine
    ↓
existing Winlator X/display path
~~~

The first milestone is control and lifecycle, not presentation purity.

## Reuse the engine, replace the shell

Meldframe should not embed Winlator's launcher/container UI as the Windows user experience.

The user-facing flow should remain:

~~~
Meldframe Start / Installer / Search
        ↓
normal AppDescriptor
        ↓
ApplicationCoordinator
        ↓
WindowsCompatibilityProvider
        ↓
WinCompat engine
~~~

Winlator's container list, shortcuts page and environment-management UI are implementation
precedents, not Meldframe shell surfaces.

A dedicated companion process/package is a plausible first deployment:

~~~
top.cmys.meldframe
        │
     Binder
        │
        ▼
top.cmys.meldframe.wincompat
~~~

Reasons include native-library isolation, crash isolation, independent runtime assets, easier
upstream rebases and clearer licence/provenance handling.

Process separation is an engineering boundary, not a claim that licence obligations disappear across
IPC. Distribution still requires a component-by-component licence review.

## CompatibilityEnvironment, not Winlator Container

The persistent Meldframe model should use a backend-neutral environment reference.

~~~
CompatibilityEnvironmentRef
├ provider
├ environmentId
├ health
├ compatibility profile
└ owned installations
~~~

Application identity must remain separate.

Avoid:

~~~
AppId.Windows(prefix03, "foo.exe")
~~~

Prefer:

~~~
AppId.Windows(productIdentity)
        ↓
InstallationRecord
        ↓
CompatibilityEnvironmentRef
~~~

This preserves Start/taskbar/pinning/default-handler identity while allowing a future application to
move between:

- one Wine version and another;
- Box64 and FEX;
- an x86_64 Wine stack and an ARM64EC-oriented stack;
- a Wine compatibility engine and a Windows ARM VM.

The runtime or prefix is provenance/execution state, not primary application identity.

## Process and window lifecycle should be proved before Wayland

A Windows process launch is not proof that a Windows application window exists.

The first serious WinCompat adapter should therefore expose separate process and window events.

A conceptual contract:

~~~
launch(...)
    → WindowsProcessRef

process events
    → started / exited / failed

window events
    → created / mapped / unmapped / destroyed
    → title / class / bounds / focus
~~~

This is particularly practical with a Winlator-derived engine because its X server already maintains
window hierarchy and lifecycle information.

An important intermediate milestone is therefore:

~~~
Wine application
    ↓
Winlator X server
    ↓
Winlator X window manager events
    ↓
WinCompat adapter
    ↓
Meldframe WindowRegistry
~~~

The image may still be presented inside one legacy Winlator display activity at this stage.

That is acceptable.

It proves the important Meldframe invariant:

> process identity and window identity are different facts.

Later presentation work can replace the renderer without replacing the application/window model.

## Presentation evolution

The recommended presentation order is deliberately incremental.

### Phase 1: existing Winlator display path

~~~
Windows application
    ↓
Wine
    ↓
Winlator X server/display
    ↓
Android surface/view
~~~

Use this to prove executable launch, graphics, audio, input, process lifecycle and application
discovery.

### Phase 2: X11 compatibility through a Meldframe window bridge

Where technically useful:

~~~
Wine X11
    ↓
XWayland / compatible bridge
    ↓
Meldframe Wayland presentation
    ↓
Android-native host window
~~~

This keeps X11 as a compatibility path without making a full X desktop the product model.

### Phase 3: Wine Wayland

The preferred long-term path is:

~~~
Win32 window
    ↓
Wine Wayland driver
    ↓
xdg_toplevel
    ↓
Meldframe WaylandPresentationProvider
    ↓
Android window/surface
~~~

At that point Linux and Windows applications can share most of the same presentation infrastructure:

~~~
Linux Qt/GTK ───────┐
                    ├─ Wayland → Meldframe → Android
Windows app → Wine ─┘
~~~

The execution backends remain different; the shell-facing window semantics converge.

The existing Winlator X/display path should remain available as a compatibility fallback rather than
being deleted for architectural purity.

## ARM64EC, FEX and Hangover direction

Meldframe should not assume that an x86_64 Wine userspace translated wholesale by Box64 will remain
the only useful Android Windows architecture.

Current ecosystem work increasingly explores combinations such as:

~~~
ARM64 / ARM64EC Wine
        +
FEX or Box64 for translated application code
~~~

Hangover is especially relevant as an architectural reference because it attempts to keep more Wine
in native ARM64 execution while translating the Windows application code that actually requires it.

This has two consequences for Meldframe:

1. The shell/provider API should not encode Box64 assumptions.
2. CPU translation should initially remain an engine-internal capability rather than becoming a
   top-level provider axis.

A future engine may internally choose among:

~~~
native ARM64 Windows application
ARM64EC path
FEX
Box64
WoW64 combinations
~~~

Only promote CPU translation into an independently plannable Meldframe dimension when real
applications demonstrate that the host needs to choose among those paths itself.

## GameNative and WinNative as integration references

Projects in the GameNative/WinNative ecosystem are useful because they explore a more componentized
Windows-on-Android stack rather than treating the Winlator UI/container model as mandatory.

Relevant ideas include:

- ARM64EC-oriented Wine/Proton builds;
- FEX integration;
- Android/Bionic compatibility fixes;
- ntsync integration;
- host clipboard and URL bridges;
- embedded Wayland compositor experiments;
- Wine Wayland sessions.

These projects should be treated as implementation references and possible component sources, not as
Meldframe product dependencies by default.

The architectural lesson is that the reusable object is the compatibility engine, not necessarily the
launcher application that originally assembled it.

## Windows application discovery

A Windows provider should convert guest/compatibility metadata into normal Meldframe
AppDescriptors.

Candidate sources include:

- explicit registration of a portable executable;
- Wine Start Menu shortcuts;
- installer-created .lnk files;
- Wine/Windows uninstall or product metadata;
- provider-specific shortcut records.

A useful installer workflow is:

~~~
snapshot known applications
        ↓
run Setup.exe / MSI
        ↓
snapshot again
        ↓
discover new launchable applications
        ↓
user chooses entries to publish
        ↓
normal AppDescriptor records
~~~

The Shell should never need to know whether an application record came from a Winlator shortcut,
Wine registry entry or Windows VM guest agent.

## Installer integration

Windows packages belong behind the generic Meldframe Installer.

### Portable executable

~~~
Foo.exe
    ↓
Windows package inspection
    ↓
select/create CompatibilityEnvironment
    ↓
register executable
    ↓
publish AppDescriptor
~~~

### Installer executable or MSI

~~~
Setup.exe / Package.msi
    ↓
WindowsInstallProvider
    ↓
WinCompat engine
    ↓
run installer
    ↓
discover resulting applications
    ↓
publish selected AppDescriptors
~~~

Environment-specific configuration remains provider-owned.

The Installer owns the user-visible transaction and application publication.

## Portal integration

Host integration should gradually move from compatibility-engine-specific bridges to Meldframe
portals.

Examples:

~~~
Wine URL open
    ↓
OpenPortal
    ↓
Meldframe Browser / Android handler
~~~

~~~
host file
    ↓
FileRef + scoped grant
    ↓
provider-exported Wine-visible path/drive
~~~

~~~
Windows notification
    ↓
compatibility bridge
    ↓
NotificationPortal
~~~

~~~
Windows file association
    ↓
AssociationPortal
~~~

The goal is not to reimplement Win32 services in Meldframe. Wine or Windows continues to implement
Win32. Meldframe only owns operations that cross the compatibility boundary into the host desktop.

## Full Windows VM path

DroidVM represents a different provider class from Wine compatibility.

Conceptually:

~~~
Meldframe
    ↓
WindowsVmProvider
    ↓
DroidVM / crosvm / QEMU
    ↓
Windows 11 ARM
~~~

This path is valuable when an application needs a real Windows kernel/userspace or behaves poorly
under Wine.

Windows on Arm can itself execute many x86/x64 user-mode applications through Microsoft's emulation
stack. This does not make x86/x64 kernel drivers portable; driver-dependent software can still require
native ARM64 Windows drivers.

The important Meldframe limitation is presentation.

A VM naturally exposes one virtual display, not one Android-native host window per Win32 HWND.

The initial VM provider should therefore be treated as a whole-environment presentation backend.

## VM guest integration

A Windows VM can still participate in Meldframe's application model through a guest agent.

Potential responsibilities:

- health and protocol negotiation;
- application discovery;
- process launch and stop;
- open URL/open file forwarding;
- clipboard;
- notifications;
- file grants/shares;
- guest shutdown;
- optional window metadata.

This resembles the Meldframe Guest Protocol but should not force Linux-specific assumptions into that
protocol.

Per-window VM integration is a separate, much harder project. It requires both semantic window
tracking and a graphics/input transport for individual Windows windows, effectively approaching a
RemoteApp-style system.

Do not make that a prerequisite for the first Windows VM backend.

## DroidVM position

DroidVM is interesting because it targets hardware-virtualized ARM guests and can run Windows ARM on
supported Android hardware.

Its trade-offs are materially different from Wine compatibility:

- requires root and suitable virtualization support;
- device compatibility is hardware/platform dependent;
- VM memory/storage overhead is substantially higher;
- a real Windows environment can improve compatibility for some workloads;
- whole-display presentation is a natural first implementation;
- host-native per-window integration is harder than Wine Wayland integration.

Recommended Meldframe position:

> experimental high-compatibility/real-Windows provider, not the default Windows application backend.

## Limbo and generic QEMU

Limbo/QEMU remains useful as a broad machine emulator and for legacy/experimental operating-system
workloads.

On ARM Android, full x86 PC emulation through QEMU TCG is generally the wrong default for a
performance-oriented Meldframe desktop.

A generic QEMU provider may eventually be useful for:

- old Windows versions;
- legacy applications;
- OS testing;
- architecture experiments;
- compatibility cases that are not latency-sensitive.

It should not drive the normal Windows application UX.

## Suggested provider hierarchy

A useful product-facing strategy is:

~~~
Windows application
      ↓
ExecutionPlanner
      ├─ WinCompat       preferred
      ├─ WindowsArmVm    compatibility fallback
      └─ QemuLegacy      legacy/experimental
~~~

The user still sees one application identity.

Provider choice should depend on application requirements, observed device capabilities, configured
environments and explicit user policy.

## Capability model

Windows support should report evidence rather than one boolean.

Candidate capability vocabulary includes:

~~~
Application architecture
- WIN_X86
- WIN_X64
- WIN_ARM64

Execution
- WINE
- NATIVE_ARM64
- ARM64EC
- BOX64
- FEX
- WINDOWS_X64_EMULATION

Graphics
- VULKAN
- TURNIP
- ZINK
- VIRGL

DirectX translation
- DXVK
- VKD3D
- WINED3D

Synchronization
- NTSYNC_KERNEL
- NTSYNC_USERSPACE
- FSYNC
- ESYNC

Presentation
- WIN_X11
- WIN_WAYLAND
- EMBEDDED_COMPAT_DISPLAY
- VM_DISPLAY

Environment
- REAL_WINDOWS_KERNEL
- WINE_COMPATIBILITY
~~~

This vocabulary does not imply that the first planner must reason independently about every item.

Initially the provider should consume most of these internally and expose only the facts required by
real planning decisions.

## Recommended implementation sequence

The Windows side track should remain subordinate to the Linux/Wayland main line, but the research
suggests a more detailed sequence than "integrate Winlator".

### W0 — component and licence audit

Review:

- Winlator;
- Wine;
- Box64/Box86;
- FEX;
- Hangover;
- GameNative/WinNative components;
- Mesa/Turnip;
- DXVK/VKD3D;
- prospective VM components.

Record source provenance, patch burden and redistribution obligations.

### W1 — WinCompat companion boundary

Define a small versioned Binder/local-protocol boundary.

Prove:

- engine health;
- environment creation/listing;
- portable executable launch;
- process exit/cancellation;
- deterministic error reporting.

### W2 — one known portable executable

Use Winlator's existing presentation stack.

Do not wait for Meldframe Wayland.

### W3 — application discovery

Convert portable registrations, shortcuts and installer-created application records into normal
AppDescriptors.

### W4 — process/window lifecycle

Expose real compatibility-window events to WindowRegistry even while rendering remains inside the
legacy display surface.

This milestone is more important than immediately achieving zero-copy or Wayland presentation.

### W5 — installer workflow

Support Setup.exe / MSI as an Installer provider workflow and prove post-install application
discovery.

### W6 — host integration

Add selected cross-boundary services:

- clipboard;
- OpenURI;
- FileRef/scoped file export;
- notifications;
- file associations.

Prefer Portal semantics where available.

### W7 — ARM64EC/FEX evaluation

Benchmark and validate real workloads.

Do not assume Box64 remains the permanent engine architecture.

### W8 — Wine Wayland

Route Windows application windows through the same Meldframe Wayland presentation infrastructure
used by Linux applications where compatibility permits.

Retain X11/legacy presentation fallback.

### W9 — Windows ARM VM provider

Add DroidVM/crosvm-style whole-environment support only after the compatibility-provider model is
stable enough that this does not distort Core.

### W10 — optional VM per-window research

Only pursue a RemoteApp-like per-window VM bridge if real workloads justify the cost.

## Acceptance criteria

A useful Windows compatibility architecture should eventually prove at least the following:

1. A portable Windows application launches through a provider without the Shell knowing Winlator
   internals.
2. Application identity survives a compatibility-environment change.
3. Process lifecycle and window lifecycle are separately observable.
4. An installed application can be discovered and published as a normal AppDescriptor.
5. A host integration such as OpenURI or file access crosses through a semantic Meldframe boundary.
6. At least two materially different Windows backends can represent the same shell-facing
   application model without backend-specific branches in Start/taskbar/Installer.

A strong long-term validation pair is:

~~~
Wine/WinCompat provider
        +
Windows ARM VM provider
~~~

## Non-goals

- Meldframe should not reimplement Wine.
- Meldframe should not make Winlator's container model its application model.
- Meldframe should not expose Box64/FEX/DXVK knobs directly through Core simply because the engine has
  them.
- The first Windows milestone does not require per-window Wayland.
- A full Windows VM does not need to masquerade as a lightweight per-application compatibility layer.
- Limbo/QEMU compatibility breadth does not justify making full CPU emulation the default path.
- Process separation is not a substitute for licence compliance.

## Research sources

Primary/current implementation references should be maintained in [the source index](../sources/README.md).

This note is research. It records current architectural conclusions and candidate implementation
directions; it does not claim that Meldframe currently ships any Windows runtime or compatibility
provider.
