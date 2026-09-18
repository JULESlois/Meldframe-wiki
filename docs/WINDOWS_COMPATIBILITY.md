# Windows Compatibility through a Winlator-derived Core

**Audience:** users, extension authors and contributors  
**Status:** proposal  
**Priority:** lower than Linux runtime and Wayland integration

Windows compatibility is a plausible Meldframe extension, but it should **not** become a second main
platform effort before the Linux/Wayland path is mature.

The proposed strategy is:

> Reuse a Winlator-derived compatibility engine behind a small Meldframe adapter instead of
> rebuilding Wine, Box64/Box86, rootfs/bootstrap, graphics translation, audio and input integration
> from scratch.

The architectural goal is not “embed Winlator UI”. It is:

> **Reuse the engine, replace the shell.**

## Why reuse Winlator

A Windows-on-Android stack already requires substantial integration work:

```
Wine
CPU translation
Linux userspace/rootfs
Mesa / Vulkan drivers
DXVK / VKD3D
display/X server
audio
input/gamepad
environment/component management
```

Reimplementing that stack now would divert effort from Meldframe's current Linux runtime, Portal and
Wayland milestones.

A Winlator-derived core lets Meldframe defer those integration costs while preserving its own app,
window, task, file and portal semantics.

## Three-layer design

```
Meldframe Shell
      │
      │ WindowsCompatibilityProvider
      ▼
Meldframe WinCompat Adapter
      │
      │ Binder / socket / versioned protocol
      ▼
Winlator-derived Core
├ Wine
├ Box64 / Box86
├ Linux/rootfs bootstrap
├ Mesa / Turnip / Zink / VirGL
├ DXVK / VKD3D
├ current display path
├ audio
├ input/gamepad
└ compatibility environment management
```

The core answers:

> How can this Windows executable actually run?

Meldframe answers:

> What application is this, how is it installed, where does it appear, how is it launched, how does
> it integrate with files/defaults/portals, and how is its window represented?

## Do not make Winlator internals part of Meldframe Core

Meldframe Core should not know about:

```
Winlator Container
Box64 presets
Wine-prefix implementation details
DXVK environment variables
rootfs layout
Winlator shortcut records
```

The first adapter can intentionally treat the Winlator-derived engine as an opaque provider.

A minimal service contract is enough:

```kotlin
interface WindowsCompatEngine {
    suspend fun inspect(): EngineCapabilities

    suspend fun createEnvironment(
        request: EnvironmentRequest
    ): EnvironmentRef

    suspend fun launch(
        request: WindowsLaunchRequest
    ): WindowsProcessRef

    suspend fun stop(process: WindowsProcessRef)

    suspend fun discoverApps(
        environment: EnvironmentRef
    ): List<WindowsAppRecord>

    suspend fun health(): EngineHealth
}
```

Only split internal concepts such as CPU translation or graphics translation into independent
providers when real workloads make that useful.

## Logical extension and deployment unit

This is a case where the logical extension probably **should** also have a physical process/package
boundary.

A likely first deployment is:

```
top.cmys.meldframe
        │
     Binder
        │
        ▼
top.cmys.meldframe.wincompat
```

The companion can own the Winlator-derived native/runtime stack and assets.

Benefits include:

- crash isolation;
- fewer native-library conflicts with the shell;
- independent update cadence;
- clearer licence/provenance tracking;
- easier upstream rebases;
- runtime assets do not bloat the base shell installation;
- experimental compatibility work does not destabilize the desktop shell.

A future implementation can change the deployment without changing the semantic provider API.

## CompatibilityEnvironment, not Container

Winlator's user-facing “Container” model should remain an implementation detail.

Meldframe should expose a backend-neutral concept:

```
CompatibilityEnvironmentRef
├ provider
├ environmentId
├ health
├ compatibility profile
└ owned applications
```

A Windows application's identity must not include a Wine prefix or Winlator container id.

Wrong:

```
AppId.Windows(prefix03, "foo.exe")
```

Preferred:

```
AppId.Windows(productIdentity)
        ↓
CompatibilityEnvironmentRef
```

That permits changing Wine versions, migrating an environment, switching translation engines, or
moving execution to another provider without changing the app shown in Start/taskbar.

## Keep Winlator's mature presentation path first

Windows compatibility should not block on Meldframe Wayland.

The earliest proof can use the compatibility engine's existing display path:

```
Windows app
    ↓
Wine
    ↓
Winlator display/X path
    ↓
Android surface/view
    ↓
Meldframe host window
```

This validates executable launch, Wine, translation, graphics, audio and input without coupling the
experiment to the unfinished Wayland bridge.

After Meldframe Wayland becomes mature, presentation can evolve:

```
Preferred
Wine Wayland
→ Meldframe Wayland bridge
→ independent Meldframe windows

Fallback
Wine X11
→ XWayland
→ Meldframe Wayland bridge

Legacy fallback
Winlator display path
```

The existing compatibility path therefore becomes a fallback rather than discarded work.

## Do not prematurely decompose the engine

The idealized long-term execution plan may eventually expose:

```
RuntimeProvider
CompatibilityProvider
CpuTranslationProvider
GraphicsTranslationProvider
VulkanProvider
PresentationProvider
AudioProvider
InputProvider
```

That is **not** the first milestone.

Initially:

```
WindowsCompatibilityProvider
└ WinlatorProvider
   └ owns the complete internal stack
```

If later there is a concrete reason to choose Box64 vs FEX, or DXVK vs another graphics stack at the
Meldframe planner level, promote that dimension then.

This follows the project's rule:

> Let a real workload force a new abstraction instead of rewriting a mature subsystem for
> architectural purity.

## Application discovery

The compatibility adapter should convert Windows/Winlator-side application records into normal
Meldframe applications.

Possible sources include:

- installer-created shortcuts;
- Start Menu entries inside the Wine environment;
- Wine integration metadata;
- uninstall/product metadata;
- portable executable registration.

The output is a normal:

```
AppDescriptor
├ AppId.Windows(...)
├ name
├ icon
├ executable reference
├ file handlers
├ compatibility requirements
└ CompatibilityEnvironmentRef
```

Start, Search, taskbar and Explorer consume that descriptor without knowing Winlator is underneath.

## Portal integration

Windows host integration should gradually map onto Meldframe portals instead of maintaining a
parallel desktop world.

Examples:

```
Wine / Windows URL open
→ OpenPortal
→ Meldframe BrowserProvider

Windows file association
→ AssociationPortal

Windows shortcut / installed app
→ LauncherPortal

Windows-visible user file
→ FileRef / FilePortal

notification bridge
→ NotificationPortal
```

The goal is not to reimplement Win32. Wine continues to implement Win32.

Only operations that cross from the Windows compatibility environment into the host desktop are
adapted to Meldframe semantics.

## Installer integration

Windows compatibility should plug into the generic Meldframe Installer rather than ship a second
launcher/installer UI.

Examples:

```
portable.exe
→ Windows package inspector
→ WindowsCompatibilityProvider
→ register normal AppDescriptor

setup.exe / package.msi
→ Windows package inspector
→ create/select CompatibilityEnvironment
→ run installer
→ discover installed applications
→ publish AppDescriptors
```

The Installer owns user-visible installation flow. The Winlator-derived core owns execution details.

## Priority and roadmap

This work is deliberately lower priority than the Linux main line.

Recommended order:

```
Main line:
Linux runtime
→ Portal/FileRef
→ Linux application discovery
→ Wayland per-window integration
→ XWayland / desktop-service integration

Windows side track:
W0 source/component/licence audit
→ W1 companion/core boundary spike
→ W2 launch one known portable EXE
→ W3 WindowsCompatibilityProvider + app discovery
→ pause if it competes with Linux Wayland

After Wayland maturity:
→ Wine Wayland integration
→ Portal integration
→ richer package/install support
→ performance and compatibility tuning
```

The first experiment only needs to prove that an opaque Winlator-derived engine can be controlled
through the Meldframe compatibility boundary. It does not need to become a finished Windows runtime.

## Upstream discipline

Keep the patch set against Winlator as small as practical.

Prefer:

```
upstream Winlator
      ↓
small engine/service patch set
      ↓
Meldframe WinCompat adapter
```

over a large Meldframe-specific rewrite.

That preserves the ability to absorb upstream fixes for Wine setup, translators, graphics, Android
compatibility, audio and controller support.

## Current status

This is an architecture proposal and research direction.

There is no claim that Meldframe currently ships a Windows compatibility engine, Winlator companion,
Wine/Wayland integration or Windows application installer.
