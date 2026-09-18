# Capabilities

**Audience:** users, extension authors and contributors  
**Status:** user/developer documentation

Meldframe uses a capability model because Android devices, OEM desktop implementations and Linux
runtimes vary too much for a simple “supported / unsupported” switch.

## Capability states

| State | Meaning |
| --- | --- |
| `Available` | Meldframe has evidence that the capability is present and working. |
| `AvailableWithSetup` | The feature can work after a setup step such as installing a runtime or granting permission. |
| `Experimental` | It works, but is not trusted enough to be treated as stable. |
| `Broken` | It was expected to work in this environment but a probe or health check observed failure. |
| `Unsupported` | Meldframe has positive evidence that the capability cannot work here. |
| `Unknown` | It has not been probed yet, or the result was inconclusive. |

`Unknown` is deliberately not the same as `Unsupported`.

This distinction allows Meldframe to avoid both false advertising and false refusal.

## Capability families

Current architecture groups capabilities roughly into:

| Group | Examples |
| --- | --- |
| Desktop | freeform mode, task observation/control/resize, per-task DPI, secondary-home behavior |
| Runtime | command execution, PRoot, chroot, services, sockets, storage, root |
| System | notifications, SystemUI bridge, LSPosed, privileged/system integration |
| Presentation | WebView, Wayland, XWayland, remote presentation, Web/Wasm features |
| Hardware | GPU, Vulkan, Turnip, audio, input devices, displays |

Extensions may contribute additional probes as their features become real.

## Detection, override and health

The effective capability is not just the result of one boot-time probe:

```
detected state
+ user override
+ runtime health
= effective capability
```

Typical user policy is:

- **Auto** — follow detection and health;
- **Force / Enabled** — attempt the feature even if automatic detection disagrees;
- **Disabled** — never use the capability.

Force exists because Android/OEM probing will never be perfect on every device. Disabled exists
because a technically working capability may still be undesirable.

## Why health matters

A runtime can be present but unhealthy.

For example:

```
Termux installed
        ≠
Termux can execute a command
        ≠
selected Linux service is healthy
```

Meldframe therefore updates effective availability as runtime/service health changes.

## Capabilities and the planner

Applications should ask for the abilities they need rather than hard-coding a particular brand or
backend.

For example, a future application might require:

```
presentation.web
websocket
clipboard
ime
gpu.webgl2
```

The planner can then choose a provider satisfying those requirements.

Similarly, Web/Wasm code should prefer asking:

> “Do I have WebGPU + IME + clipboard?”

instead of:

> “Am I running in this exact WebView implementation?”

## Capabilities are not permissions

A capability describes whether Meldframe can provide something.

A permission describes whether a particular extension is **allowed to use** it.

For example:

```
capability: clipboard.read is technically available
permission: extension X may still be denied clipboard.read
```

The extension `PermissionBroker` exists to keep those concerns separate.

## Device profiles

Meldframe prefers probes over vendor conditionals.

Instead of putting `if Samsung`, `if Xiaomi` or `if WSA` throughout Core, a known device may
have a data-driven profile that overrides unreliable detection.

The rule is:

```
probe first
→ profile override when needed
→ user override last
```

## Runtime isolation is related but distinct

A runtime may support the same commands while offering very different isolation.

PRoot, Android sandboxing, chroot, VM and remote execution are not interchangeable security
boundaries. The roadmap therefore treats runtime isolation/trust as another planner input rather than
pretending “can execute Linux command” is enough.

## For bug reports

A useful capability bug report should include the state **and reason**.

Examples:

```
runtime.termux = AVAILABLE_WITH_SETUP
reason = RUN_COMMAND permission denied

desktop.taskControl = BROKEN
reason = provider returned incomplete task snapshot

presentation.wasmThreads = AVAILABLE_WITH_SETUP
reason = SharedArrayBuffer / cross-origin isolation unavailable
```

This is much more actionable than “feature missing”.
