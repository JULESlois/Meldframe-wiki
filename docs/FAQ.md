# Frequently asked questions

**Audience:** users and evaluators  
**Status:** user documentation

This FAQ is intentionally conservative. Meldframe is under active development, so a roadmap direction is not described as a shipping feature.

## Is Meldframe just an Android launcher?

No, although it started from a desktop-launcher codebase.

A conventional launcher mostly discovers Android packages and launches Activities. Meldframe is being refactored around a common application identity, runtime providers, presentation backends, capability planning and backend-neutral window state so Android, Web and Linux-oriented workloads can participate in one desktop model.

The launcher/shell UI remains important, but it is no longer the whole architecture.

## Is Meldframe an operating system or custom ROM?

No. The main product is an Android application/shell and composition layer. Android remains responsible for core mechanisms such as tasks/windows, SurfaceFlinger, input/IME, WebView and platform security.

More privileged integrations may be possible on rooted, Shizuku-enabled, system-app or custom-ROM environments, but Meldframe should remain useful without requiring a replacement OS.

## Does Meldframe run Linux applications?

There are several separate milestones here, and treating them as one “Linux support” switch is misleading.

**Implemented:** Meldframe can use the current external-Termux provider for command/service workloads. It can also parse registered Linux `.desktop` entries into the common application catalogue and list those entries in Start. This discovery path has been verified with real WPS Office and Cylheim desktop entries.

**Not yet implemented as a product path:** an `AppId.Linux` entry does not currently have a Linux GUI execution/presentation backend. Selecting such an entry therefore must not be interpreted as native-style Linux GUI support.

**Research evidence:** the M3 whole-display experiment has run a real Linux GUI application under a nested compositor and demonstrated that rendered frames reach an RFB connection. The browser-side noVNC test path did not render those frames, and per-window Wayland integration has not been implemented. This is useful engineering evidence, not a shipping Linux desktop feature.

See [Linux applications](LINUX_APPS.md) and [Runtimes](RUNTIMES.md) for the exact boundary.

## Why can a Linux application appear in Start but not launch?

Discovery and execution are deliberately separate.

A registered container can contribute valid `.desktop` metadata to the application catalogue, so Meldframe can know the application's identity, localized name and intended argv before a compatible GUI backend exists. Current Linux entries therefore prove catalogue integration, not launchability. Meldframe should refuse an unsupported launch explicitly rather than pretending that discovery implies execution support.

The current Linux icon is also a generic placeholder because resolving a `.desktop` `Icon` normally requires access to the guest's icon themes; that integration is not implemented yet.

## Does the Linux GUI proof of concept mean Wayland support is finished?

No.

The experiment proves only specific links in the chain: a real Linux GUI application can run, the compositor can render it, and the resulting frame can reach the RFB wire. It does not establish a production presentation backend, Start-to-GUI launch path, per-window `xdg_toplevel` integration, input/IME integration, clipboard integration, accelerated buffer sharing or broad application compatibility.

A failed or successful research harness is evidence about that harness. It is not automatically a product compatibility claim.

## Do I have to use Termux?

For the current first Linux runtime path, external Termux is the implemented provider used by development builds and tests.

Architecturally, no. Meldframe is intentionally based on `RuntimeProvider`, with a future curated Meldframe Runtime companion, chroot, AVF and SSH/remote providers among the planned/research directions. An application should not become permanently identified with Termux.

## What is Meldframe Runtime?

It is a planned dedicated companion runtime intended to reduce setup friction and give Meldframe a predictable Unix/Linux service environment.

It should be understood as one provider, not a private fork that every Meldframe feature is forced to depend on. External Termux should remain a useful compatibility/provider path where practical.

## Does Meldframe require root?

No for the basic architecture and normal Android/application paths.

Some deeper desktop/task/SystemUI integrations inherently require more authority than an ordinary APK receives. Meldframe models those as optional capability/provider tiers rather than making root a universal requirement.

## Is Wayland support finished?

No. Per-window Wayland integration is a roadmap target.

The planned progression deliberately starts with correct `xdg_toplevel` lifecycle and copied/shared-memory frames, then moves toward damage-aware copy, dma-buf and AHardwareBuffer/SurfaceControl fast paths. This avoids treating zero-copy graphics as a prerequisite for proving the window model.

## What is the guest capability probe, and does Meldframe use it to choose a runtime today?

`GuestCapabilityProbe` is implemented and unit-tested groundwork for asking a particular guest environment about facts such as libc, PRoot, Wayland/X11/XWayland and graphics availability. Its verdicts intentionally distinguish `PRESENT`, `ABSENT` and `UNKNOWN`; an inconclusive probe is not converted into “unsupported”.

It is **not currently wired into a production RuntimeProvider or planner path**. Therefore the existence of the probe does not mean Meldframe already performs automatic live guest selection. It is a prerequisite for later runtime planning work.

The distinction matters because two environments reached through the same host app can have different capabilities—for example a native Termux userspace and a Debian userspace under PRoot.

## Is Meldframe trying to replace Android's window manager?

No. The current architectural direction is the opposite: use Android's strongest existing task/window/compositor mechanisms and keep Meldframe focused on application identity, policy, planning, compatibility and cross-runtime composition.

## Is Code a Linux GUI application?

Not in the conventional sense.

The current Code direction uses code-server as a Linux service and Android WebView as presentation. Meldframe supervises the service and opens one application window when the backend is healthy.

This is an example of why execution and presentation are separate concepts.

## Is the Terminal just Termux embedded in Meldframe?

No. Terminal has its own Meldframe domain model and frontend. The current path uses xterm.js/WebView with a pluggable `TerminalTransport`, with ttyd serving as the first practical transport to a selected runtime.

This design is intended to let the same Terminal frontend target different runtimes later.

## Why does Meldframe use WebView at all?

WebView is one presentation technology, not the foundation of the whole shell.

It is useful for workloads already designed around browser protocols, including code-server, xterm.js and Web/Wasm applications. Android-native tasks and future Wayland presentation remain separate options.

## Is WebAssembly a replacement for Android UI or Linux?

No. Meldframe treats WebAssembly/WASI as another runtime/capability technology. A Wasm extension may be valuable for portable or sandboxed logic, but the project does not assume every UI or application should be rewritten in Wasm.

## Can I install third-party Meldframe plugins now?

There is not yet a stable public external-plugin package format.

Typed extension contributions already exist and built-in extensions are used to validate the architecture. External manifests, distribution, version negotiation, permissions, signing/trust and cross-process protocol still need to be stabilized before the project can make a compatibility promise to third-party authors.

## Why not simply make every plugin a separate APK?

Because API modularity and process isolation are different decisions.

A logical extension may be built into the main APK, run in another process, ship as a companion APK, execute in Linux or use a Wasm sandbox. Forcing every semantic boundary to become an IPC/release boundary would add startup, serialization, lifecycle and version-skew costs without necessarily improving the design.

## What does `AvailableWithSetup` mean?

Meldframe has evidence that the feature can work, but a required setup step is missing — for example an external runtime or permission.

It is different from `Unsupported`, `Broken` and `Unknown`. See [Capabilities](CAPABILITIES.md).

## Why can a feature be `Unknown`?

Because Meldframe prefers admitting uncertainty to guessing.

An unrun or inconclusive probe does not prove that a device is unsupported. Conversely, a version string or OEM name is not strong evidence that a feature works.

## Why can a runtime be installed but not ready?

Presence is not health. Termux can be installed while external command execution is disabled, permission is missing, setup is incomplete or a service cannot start.

Meldframe therefore distinguishes package detection from successful command/service health checks.

## Is ChromeOS the model Meldframe is copying?

ChromeOS is an important architectural reference, not the product template.

App Service, Crostini guest integration and Sommelier demonstrate useful solutions to heterogeneous apps and Linux integration. Meldframe's research question is somewhat different: how one logical application can be composed across replaceable execution and presentation environments while Android remains the host OS.

See [ChromeOS, Android and Meldframe](../research/chromeos-android-meldframe.md).

## Which documentation is authoritative?

For current behavior, the implementation repository's code, tests and architecture contracts are authoritative.

This wiki explains the product, teaches concepts and records research/decisions. If wiki prose gets ahead of implementation, that is a documentation bug rather than a new product guarantee.

## Where should I start?

For a first read:

1. [What is Meldframe?](INTRODUCTION.md)
2. [Features and current status](FEATURES.md)
3. [Installation](INSTALLATION.md)
4. [Using Meldframe](USAGE.md)
5. [Troubleshooting](TROUBLESHOOTING.md)

For technical concepts, continue with [Glossary](GLOSSARY.md), [Capabilities](CAPABILITIES.md), [Extensions and plugins](EXTENSIONS.md), and [Architecture overview](ARCHITECTURE.md).
