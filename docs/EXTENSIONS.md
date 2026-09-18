# Extensions and plugins

**Audience:** users, extension authors and contributors  
**Status:** user/developer documentation

Meldframe is designed so many features can be added as **typed extensions** rather than special cases
inside the shell.

“Plugin” is the user-friendly term. In the architecture, an extension contributes one or more
specific capabilities to the shell.

## What can be an extension?

Examples include:

- an application such as Terminal, Explorer or Browser;
- a runtime provider such as Termux, AVF, SSH or a future Meldframe Runtime;
- a presentation provider such as Wayland or a Web/Wasm host;
- a service-backed application adapter such as code-server;
- a compatibility layer such as ElectronCompat or a Chromium/Playwright bridge;
- a capability probe;
- a future file provider;
- a future shell/personality surface.

The important point is that these are **different contribution types**. Meldframe does not use a
single giant `Plugin.doEverything()` interface.

## Contribution model

Current contribution points include:

| Contribution | Purpose | Current state |
| --- | --- | --- |
| App | Adds an application descriptor | implemented |
| Runtime | Adds a RuntimeProvider | implemented |
| Terminal | Adds terminal transports/profiles | implemented |
| ServiceApp | Adds a supervised service-backed application | implemented |
| CapabilityProbe | Adds evidence-based capability detection | implemented |
| PresentationBackend | Adds a presentation provider | implemented |
| WasmApp | Adds a Wasm application descriptor/presentation requirements | implemented |
| FileProvider | Adds filesystem/source integration | planned with unified FileRef |
| ShellSurface | Adds personality/shell surfaces | planned |

## Existing examples

### Terminal

Terminal proves that one extension can contribute several unrelated typed pieces:

```
Terminal extension
├ App
├ Runtime
└ Terminal transport/profile
```

The application catalogue consumes the App contribution; the runtime catalogue consumes Runtime; the
Terminal subsystem consumes the terminal contribution.

### Code

Code is a `ServiceApp`:

```
Code descriptor
+
code-server ServiceSpec
+
Web presentation
```

The generic service supervisor and WebView host do the work. The shell should not contain
code-server-specific behavior outside the extension/adapter.

### Wasm

The Wasm extension contributes:

```
CapabilityProbe
PresentationBackend
WasmApp
```

This lets the shell discover actual WebAssembly features and plan Wasm applications without turning
Wasm into a hard-coded global assumption.

## Logical extension does not mean separate APK

This is a core design rule.

A plugin/extension is a **semantic boundary**. Its physical deployment is a separate choice.

Possible deployments include:

```
built into the main APK
separate Android APK/service
native or Linux process
Wasm/WASI sandbox
remote component
```

For example, Terminal can remain built-in while the API stabilizes; a GPL Wayland component may need
a process boundary; an untrusted formatter could run in a Wasm sandbox.

## Trust levels

The current architecture names three trust classes:

| Trust | Meaning |
| --- | --- |
| Built-in | Compiled as part of the shell; receives only the permissions it declares as part of the trusted product. |
| Process | Separate APK/native/Linux process; no grants by default. |
| Sandboxed | Wasm/WASI-style sandbox; no grants by default. |

Only the built-in path is mature today. External third-party extension installation is a future
milestone.

## Permissions

Declaring a permission is not the same as receiving it.

A future extension might declare:

```
files.read
files.write
clipboard.write
runtime.exec
runtime.services
window.create
```

The `PermissionBroker` decides whether that extension actually receives the capability.

Privileged bridges should expose semantic operations such as:

```
file.read(FileRef)
runtime.exec(RuntimeRef, Command)
window.open(AppId)
clipboard.write(...)
```

rather than raw `Android Context`, arbitrary Intents, unrestricted shell execution or direct
`SurfaceControl.Transaction` access.

## Package format status

Meldframe **does not yet promise a stable external plugin package format**.

That is deliberate. The project is first validating the contribution model using unlike real
extensions before freezing:

- manifest schema;
- distribution/package format;
- cross-process wire protocol;
- version compatibility rules;
- third-party permission UX;
- signing/trust rules.

This avoids publishing an API that immediately has to be broken.

## Versioning direction

External extensions will need explicit version negotiation, conceptually including:

```
manifestVersion
hostApiVersion
minHostVersion
requiredCapabilities
optionalCapabilities
permissions
contributions
```

First-party built-ins may move with the host build, but third-party extensions must not rely on that.

## For extension authors

Until the external SDK is declared stable, treat the public extension documentation as an
architecture preview rather than a compatibility guarantee.

The strongest current rule is already stable in spirit:

> depend on Meldframe Core contracts, not MainActivity, a particular WebView instance, a recovered UI
> class or a specific runtime implementation.

See also:

- [Capabilities](CAPABILITIES.md)
- [Architecture overview](ARCHITECTURE.md)
- [Development principles](../development/PRINCIPLES.md)
