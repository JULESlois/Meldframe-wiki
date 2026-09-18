# Meldframe glossary

**Audience:** users, extension authors and contributors  
**Status:** user/developer documentation

Meldframe deliberately separates concepts that ordinary Android launchers often collapse together. This glossary defines the terms used across the public documentation. It describes the current architecture; entries marked **planned** describe roadmap concepts rather than finished APIs.

## Application terms

### AppId

The stable identity of a logical application inside Meldframe. An AppId should not change merely because the application is launched through a different runtime or presentation backend.

For Android applications, identity must be able to distinguish profiles/users rather than assuming a package name is globally unique.

### AppDescriptor

Metadata describing a logical application to the common application catalogue: identity, name/icon information and the information required for launch planning. UI surfaces consume descriptors instead of directly knowing how Android, Linux, Web or service-backed applications are discovered.

### ApplicationCoordinator

The common launch entry point. Start, Search, taskbar and other surfaces should submit a launch request rather than implementing backend-specific launch logic themselves.

### ExecutionPlan

The selected composition used to run an application. A plan can combine an execution provider, presentation provider, runtime and compatibility adapter.

A single logical app may eventually have several candidate plans. This is a design goal, not a claim that every current app already has multiple interchangeable plans.

## Runtime terms

### RuntimeProvider

A provider capable of creating or describing places where work can execute. External Termux is the first real provider path. Meldframe Runtime, chroot, AVF and SSH are planned/research provider families.

A RuntimeProvider is not itself an application.

### RuntimeInstance

One concrete runtime target selected from a provider. Profiles and applications should target runtime instances rather than hard-coding “Termux” into unrelated UI/domain models.

### Runtime health

Evidence that a runtime can actually perform the operation Meldframe needs. “Termux is installed” is therefore weaker than “a command round-trip succeeded”.

### Meldframe Runtime

**Planned.** A dedicated companion runtime intended to make Linux/Unix integration easier and more predictable than requiring users to assemble an external environment manually. It is one RuntimeProvider, not the definition of Linux support as a whole.

### Meldframe Guest Protocol

**Planned/proposed.** A versioned semantic host↔guest protocol for operations such as health, exec, PTY, services, app discovery, file grants and endpoint publication. The goal is to avoid binding Meldframe Core to one Termux intent or one Linux container technology.

### PRoot

A userspace compatibility mechanism commonly used to provide Linux filesystem/distribution environments without root. In Meldframe architecture it must not be treated as a strong security boundary.

### AVF

Android Virtualization Framework. AVF-backed Linux is a roadmap RuntimeProvider direction for devices/platform versions where the required APIs and integration are available. It is not a current universal Meldframe runtime requirement.

## Presentation and window terms

### PresentationBackend / PresentationProvider

The mechanism that presents an application to the user. Examples include an Android task, a Meldframe-owned view/WebView and, in the roadmap, Wayland or remote presentation.

Execution and presentation are intentionally independent: a Linux service can be presented by Android WebView.

### WindowRegistry

Meldframe's backend-neutral record of live window/task facts. It does not mean Meldframe owns Android's physical compositor.

### Wayland bridge

**Roadmap.** The planned integration that maps Linux `xdg_toplevel` windows into independent Meldframe/Android window semantics. The target is not to place an entire Linux desktop inside one Android window.

### SurfaceControl fast path

**Roadmap/experimental direction.** A later graphics path intended to reduce copying after correct Wayland lifecycle/input/resize semantics are proven. Zero-copy is not a prerequisite for the first Wayland multi-window milestone.

## Capability terms

### Capability

Evidence that Meldframe can provide a technical ability in the current environment. Capability state is richer than a boolean: `Available`, `AvailableWithSetup`, `Experimental`, `Broken`, `Unsupported` or `Unknown`.

See [Capabilities](CAPABILITIES.md).

### Capability probe

A component that gathers evidence about a capability. Meldframe prefers real probes and health checks over inferring support from a device brand or version string alone.

### Device profile

A data-driven correction for devices/environments where generic detection is insufficient or misleading. Profiles should be exceptions around the probe model, not a replacement for it.

### Permission

Authorization for an extension to use an ability. A capability being available does not imply every extension is allowed to use it.

### PermissionBroker

The architecture component responsible for extension authority. The external third-party permission UX and package system are not yet stable public APIs.

## Extension terms

### Extension / plugin

A typed contribution to Meldframe. “Plugin” is the user-facing term; “extension” is commonly used in architecture documentation.

An extension can contribute an App, Runtime, Terminal transport/profile, ServiceApp, CapabilityProbe, PresentationBackend or WasmApp. Future contribution types include FileProvider and shell/personality surfaces.

### Contribution

One typed feature supplied by an extension. Meldframe uses multiple narrow contribution types instead of a single catch-all plugin interface.

### Logical extension

A semantic/API boundary. It does **not** imply a separate APK, process or release train.

### Deployment unit

Where an extension physically runs or ships: for example in the main APK, another Android process/APK, a native/Linux process, a Wasm sandbox or a remote component. Deployment is chosen separately from logical extension design.

### Built-in extension

An extension compiled and shipped with Meldframe. Built-ins are currently the mature path used to validate the extension architecture before freezing an external plugin format.

### External plugin

**Planned.** A third-party extension installed/distributed independently of the main build. Meldframe does not currently promise a stable external plugin package, manifest or compatibility ABI.

## Service and compatibility terms

### ServiceSupervisor

The generic lifecycle/health layer for long-running service-backed applications. Code/code-server is the first important workload validating this abstraction.

### Service-backed application

An application whose backend is a supervised service while its user-facing presentation is supplied elsewhere. Code demonstrates the pattern with code-server plus Android WebView.

### Browser Broker

**Research/roadmap.** A host-side browser capability that could let Linux programs use Android WebView/Chromium for URL presentation or automation rather than requiring a complete Linux Chromium stack.

### ElectronCompat

**Research/roadmap.** A compatibility direction in which compatible Electron applications could keep a Linux/Node backend while moving presentation to an Android WebView/Chromium surface. It is not a general-purpose Electron compatibility guarantee.

## Documentation status terms

### Implemented

Code exists in the implementation repository. This does not automatically mean it has broad real-device validation.

### Verified

A concrete path has been exercised successfully in a documented test environment. Verification on one environment is not proof of universal compatibility.

### Experimental

The feature works in at least some relevant conditions but is not considered sufficiently stable or portable.

### Planned / roadmap

An intended direction with enough architectural commitment to appear in the roadmap. It is not a current product feature.

### Research / idea

A possibility under investigation. Research notes are evidence and reasoning, not promises.

## Related reading

- [What is Meldframe?](INTRODUCTION.md)
- [Features and current status](FEATURES.md)
- [Capabilities](CAPABILITIES.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
