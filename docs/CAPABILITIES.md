# Capabilities

**Audience:** users, extension authors and contributors  
**Status:** implemented core model with partially wired probes; some capability families remain roadmap

Meldframe uses a capability model because Android devices, OEM desktop implementations and Linux
runtimes vary too much for a single “supported / unsupported” switch.

> **Source-of-truth rule:** this page explains the model and the capabilities currently backed by
> implementation. It is not a promise that every named capability is probed, selectable, or usable.
> The implementation repository, tests and device evidence decide that.

## What is implemented today

The distinction between the **capability model** and a **working feature** matters.

| Piece | Current status |
| --- | --- |
| `CapabilityState`, health and user-override types | Implemented in Core |
| `CapabilityGraph` effective-state resolution | Implemented and unit-testable Core logic |
| `AndroidCapabilityProbe` | Implemented for a deliberately limited set of Android facts |
| `CapabilityAwarePlanner` | Implemented; refuses a plan only when its required presentation capability is *known* unusable |
| Extension `CapabilityProbe` contribution | Implemented typed contribution with a real consumer |
| `GuestCapabilityProbe` | Implemented and unit-tested, but **not yet wired to a runtime provider/planner path** |
| Complete automatic detection of every capability listed below | **Not implemented** |
| A mature user-facing compatibility-test/override UI for every capability | **Not implied by the Core model** |

In particular, the existence of `presentation.wayland`, `runtime.glibc`, or another capability key does
not prove that Meldframe can currently provide that feature.

## Capability states

| State | Meaning |
| --- | --- |
| `Available` | Detection has evidence that the capability is present and working. |
| `AvailableWithSetup` | It may work after a setup step such as installing a runtime or granting access. |
| `Experimental` | It works, but is not trusted enough to be treated as stable. |
| `Broken` | It was expected to work in this environment but a probe or health check observed failure. |
| `Unsupported` | Meldframe has positive evidence that it cannot work here. |
| `Unknown` | It has not been probed, or the result was inconclusive. |

`Unknown` is deliberately not `Unsupported`. It is also not `Available`.

That produces two different questions:

- **May Meldframe advertise/rely on it?** `usable` is false for `Unknown`.
- **Does Meldframe have evidence that it will fail?** `isKnownUnusable` is also false for `Unknown`.

This prevents an inconclusive probe from either advertising an unsupported feature or blocking a path
that previously worked.

## What Android currently probes

`AndroidCapabilityProbe` intentionally reports only facts it can establish with useful confidence:

- freeform-window support;
- WebView availability;
- Vulkan availability;
- attached displays;
- whether a known runtime-host package is installed;
- Android task-provider admission states for observation, control and resize.

A runtime host being installed is only `AvailableWithSetup`: package presence does not prove that a
command can execute. Likewise, an ordinary Android application being able to launch an Activity or
inspect its own tasks is not evidence of desktop-wide task authority.

Several tempting probes intentionally remain `Unknown` until an appropriate backend can prove them.
Examples include root/LSPosed/privileged-app state and WM-Shell-dependent operations such as
per-task density or custom decoration. Wayland/XWayland are not inferred from Android device
properties merely because capability keys exist for them.

## Guest Linux capability probing

The newer `GuestCapabilityProbe` models facts about a *particular guest environment*, rather than
asking only whether Termux is installed. This distinction is necessary because a native Termux shell
and a Debian or Alpine guest under PRoot can have different libc, binaries and presentation options.

The probe's verdict logic is implemented and tested, including two non-obvious rules:

1. `ldd --version` exiting successfully does **not** prove glibc; musl can also answer it, so libc is
   classified from the output.
2. XWayland by itself is **not** a presentation backend. Without a Wayland compositor there is
   nowhere for the X11 surface to be presented.

However, **nothing in the production runtime path calls this probe yet**. Its results must not be
read as live Linux compatibility information in the current product. Provider wiring is a later
runtime milestone.

## Capability families

The architecture groups capability keys into these families:

| Group | Examples |
| --- | --- |
| Desktop | freeform mode, task observation/control/resize, per-task DPI, secondary-home behavior |
| Runtime | command execution, libc/runtime facts, PRoot, chroot, services, sockets, storage, root |
| System | notifications, SystemUI bridge, LSPosed, privileged/system integration |
| Presentation | WebView, Wayland, XWayland, remote presentation, Web/Wasm features |
| Hardware | GPU, Vulkan, Turnip, audio, input devices, displays |

This is a taxonomy, not a support matrix. See [Features](FEATURES.md) and
[Compatibility](COMPATIBILITY.md) for user-facing status.

## Detection, override and health

Core resolves three inputs:

```
detected state
+ user override
+ live health
= effective capability
```

The model supports:

- **Auto** — follow detection and health;
- **Force enabled** — attempt use despite the detected state, unless health is currently failing;
- **Disabled** — do not use the capability.

These are implemented Core semantics. Do not infer from them that every capability currently has a
finished Settings control.

Health is deliberately separate from detection. A runtime can have been detected successfully and
later stop responding; a failing health value makes it effectively unusable without pretending the
original detection result changed.

## Capabilities and planning

The intended architecture is for applications to ask for abilities rather than hard-code a vendor or
backend. `CapabilityAwarePlanner` is an early real consumer of this model.

There is an important conservative rule: shell-owned presentation paths such as Android tasks and
Meldframe views are not capability-gated merely because an unrelated probe is unknown. For optional
backends, positive evidence of failure may reject a plan; lack of evidence alone should not.

Broader examples such as selecting a future provider from WebGPU, clipboard, IME or Wayland
requirements describe the direction of the planner, not a claim that all of those negotiations are
implemented today.

## Capabilities are not extension permissions

A capability answers whether Meldframe can provide something. An extension permission answers
whether a particular extension may use something.

Core already contains `PermissionBroker`, manifest declarations and per-extension grants. That does
**not** make Meldframe a finished third-party plugin sandbox: external package loading, stable SDK
policy and complete host-API permission enforcement remain separate work. See
[Extensions and plugins](EXTENSIONS.md).

## Device profiles

The architectural rule is to prefer probes over scattered vendor conditionals. A known-device profile
may eventually override unreliable detection with data rather than teaching Core to branch on OEM
names.

Treat this as an engineering policy, not evidence that Meldframe has a comprehensive database of OEM
profiles today.

## Reading capability diagnostics

When diagnostics expose a capability, keep the detected state, health, override and reason together.
For example:

```text
runtime.termux = AVAILABLE_WITH_SETUP
reason = runtime host installed; execution not yet proven

presentation.example = UNKNOWN
reason = probe did not establish a result
```

Do not rewrite `UNKNOWN` as “unsupported”, and do not rewrite `AVAILABLE_WITH_SETUP` as “working”.
Those shortcuts erase exactly the information the model exists to preserve.

For Linux GUI issues, also distinguish **discovery**, **execution** and **presentation**. A `.desktop`
entry appearing in Start proves discovery; it does not prove that a Linux GUI launch backend or
Wayland/XWayland presentation path exists.

## For bug reports

A useful capability report includes:

- the capability key;
- detected state;
- effective state/reason when available;
- current health;
- any user override;
- device/runtime identity and the operation that failed.

A report saying “feature missing” is much less useful than one that distinguishes “not probed”,
“needs setup”, “known unsupported” and “was expected to work but failed”.
