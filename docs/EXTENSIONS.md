# Extensions and plugins

**Audience:** users, extension authors and contributors  
**Status:** typed built-in extension model implemented; third-party installation/packaging not implemented

Meldframe is designed so many features can be added as **typed extensions** rather than special cases
inside the shell. “Plugin” is a useful user-facing term, but it needs one important qualification:
**Meldframe does not currently have a public installable-plugin ecosystem.** The extension model that
exists today is an internal/core contract used by built-in features.

This page therefore separates what can be used today from the architecture intended to support
external extensions later. The implementation repository remains the source of truth if this page
lags behind it.

## What exists today

The current registry accepts an `ExtensionManifest` containing typed contributions. Consumers ask for
the contribution type they understand; there is deliberately no single `Plugin.doEverything()`
interface.

| Contribution | Consumer / purpose | Current state |
| --- | --- | --- |
| `App` | Adds an `AppDescriptor` to the application catalogue | implemented |
| `Runtime` | Adds a `RuntimeProvider` | implemented |
| `Terminal` | Adds terminal transports/profiles | implemented |
| `ServiceApp` | Adds an app descriptor, supervised `ServiceSpec` and path to present | implemented |
| `CapabilityProbe` | Adds evidence-based capability detection | implemented |
| `PresentationBackend` | Adds a presentation provider used during planning | implemented |
| `WasmApp` | Adds a Wasm app descriptor and presentation requirements | implemented |
| file-provider contribution | Future filesystem/source integration | planned; do not treat as an extension API yet |
| shell-surface contribution | Future personality/shell integration | planned; do not treat as an extension API yet |

The distinction in the last two rows matters: an architectural slot mentioned in design documents is
not an API merely because it has a name.

## Built-in extensions that exercise the model

### Terminal — implemented and device-verified

Terminal demonstrates one manifest contributing several unrelated typed pieces:

```
Terminal extension
├ App
├ Runtime
└ Terminal transport/profile
```

The application catalogue consumes the app contribution, the runtime catalogue consumes the runtime
provider, and Terminal consumes the transport/profile. The implementation repository records the
extension in diagnostics as `top.cmys.meldframe.terminal 0.1.0` and has verified the ttyd-backed path
on WSA.

This does **not** mean every terminal transport discussed by the project is implemented. The current
end-to-end verified transport is ttyd; direct Termux PTY, Runtime Agent, local PTY and SSH transports
remain separate future work. See [Terminal](TERMINAL.md).

### Code — implemented and device-verified

Code is a `ServiceApp` contribution containing its descriptor, a code-server `ServiceSpec`, and the
path to open. The generic `ServiceSupervisor`, gateway and WebView host provide lifecycle and
presentation; Code-specific startup knowledge stays in the extension.

The current path has been verified on WSA with a real Termux code-server service, including health
checking, workbench/WebSocket/extension-host connectivity and the integrated title-bar path. The
service-backed popup route was also reverified on 2026-09-17.

The security boundary is narrower than simply exposing an unauthenticated loopback page: the current
host uses a per-service token proxy with an ephemeral port and a per-process session cookie. There is
still a documented gap: the backend service's own loopback listening port remains reachable until a
future Unix-socket backend removes it. See [Code and service-backed applications](CODE.md).

### Wasm — implemented and device-verified

The Wasm extension contributes:

```
CapabilityProbe
PresentationBackend
WasmApp
```

Its capability probe executes real WebAssembly tests rather than inferring support from a Chromium
version. On WSA on 2026-09-17, WebView 118.0.5993.111 executed the basic module and validated SIMD;
atomic instructions validated while SharedArrayBuffer/cross-origin isolation were absent, so Threads
correctly remained `AVAILABLE_WITH_SETUP`. WASI likewise remained `AVAILABLE_WITH_SETUP` because the
WebView provider is not a WASI host.

Those observations are evidence from one environment, not a compatibility promise for every Android
WebView version.

## What users cannot do yet

There is currently no supported workflow to download an arbitrary Meldframe plugin, install it from a
file/store, grant it third-party permissions and rely on a stable public SDK. In particular, the
project has **not** frozen:

- an external manifest/package format;
- an extension repository or distribution mechanism;
- a cross-process extension wire protocol;
- host/extension compatibility negotiation;
- third-party signing and trust policy;
- third-party permission UX.

If a document, issue or roadmap sketch describes one of these, read it as design/research material
unless the implementation repository contains the corresponding consumer and execution path.

## Registry behavior today

`ExtensionRegistry.install(manifest, trust)` is the current lifecycle primitive. It rejects duplicate
extension IDs. It can also reject a manifest when a required capability is positively known to be
unusable; an unprobed capability is not silently converted into a negative verdict.

The `PermissionBroker` records grants per extension and answers them per call. A permission declaration
is not itself a grant.

For built-ins, manifests are currently installed by the Android composition root. That is a manifest
**loader for trusted built-ins**, not evidence that arbitrary manifest files can be loaded from disk.

## Trust levels: vocabulary versus implementation

The architecture names three trust classes, but only one is implemented today:

| Trust | Intended meaning | Product status |
| --- | --- | --- |
| `BUILT_IN` | Compiled into the shell | implemented and used |
| `PROCESS` | Separate APK/native/Linux process reached over a protocol | design target; not an installable extension path today |
| `SANDBOXED` | Wasm/WASI-style isolated component | design target; not an installable extension path today |

This is a common source of overstatement. The existence of enum/vocabulary for a trust boundary does
not prove that loading, protocol negotiation, permission UX or lifecycle for that boundary exists.

## Logical extension does not mean separate APK

An extension is first a **semantic boundary**. Physical deployment is a separate decision. A built-in
feature can already obey Core contracts and be an extension without becoming a separately installed
APK.

Possible future deployments include a separate Android service/APK, native or Linux process,
Wasm/WASI sandbox, or remote component. These are architectural options rather than currently
supported package types.

This distinction is useful for licensing and fault isolation as well as code structure: for example,
a future copyleft Wayland component may warrant a process boundary, while a small trusted first-party
adapter may remain built into the host.

## Permission design

Privileged extension operations should be semantic rather than exposing unrestricted platform
objects. The design direction is closer to:

```
file.read(FileRef)
runtime.exec(RuntimeRef, Command)
window.open(AppId)
clipboard.write(...)
```

than handing an extension an Android `Context`, unrestricted Intent access, an arbitrary shell, or a
raw `SurfaceControl.Transaction`.

Names such as `files.read`, `runtime.exec` or `window.create` that appear in design examples should
not be interpreted as a frozen public permission namespace until an external extension SDK actually
ships.

## Guidance for contributors

When adding a first-party extension now:

1. contribute the narrow typed contracts consumed by the relevant subsystem;
2. keep application/runtime-specific startup knowledge in the extension or adapter;
3. depend on Meldframe Core contracts rather than `MainActivity`, a particular WebView instance,
   recovered UI classes or a concrete runtime implementation;
4. treat a new contribution kind as justified only when a real consumer exists;
5. document device verification separately from unit-tested or model-only behavior.

There are still known compromises. The recovered `WindowsHost` is currently the only window host, so
an extension window is mapped by the Android host to an app ID. Built-in manifests are also installed
from the composition root rather than discovered by a general package loader. These are migration
boundaries, not features to copy into a future external SDK.

## Versioning direction

External extensions will eventually need explicit compatibility negotiation, potentially covering
manifest version, host API version, minimum host version, capabilities, permissions and contribution
versions. None of those fields should be treated as a stable schema yet.

Freezing a package format before a second independently deployed implementation exists would turn
current implementation accidents into compatibility obligations. The project is intentionally using
Terminal, Code and Wasm as unlike built-in cases before defining that boundary.

## Related documentation

- [Terminal](TERMINAL.md)
- [Code and service-backed applications](CODE.md)
- [Capabilities](CAPABILITIES.md)
- [Architecture overview](ARCHITECTURE.md)
- [Development principles](../development/PRINCIPLES.md)
