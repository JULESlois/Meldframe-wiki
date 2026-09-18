# Meldframe Wiki

> **Meldframe is a composable desktop runtime and shell for Android.**

Meldframe brings Android, Web and Linux-oriented workloads into one desktop model without requiring
all of them to run or render the same way.

An application can keep one user-visible identity while its execution, presentation, runtime and
compatibility layers are selected independently:

```
one app
  ↓
Meldframe planner
  ├─ Android execution
  ├─ Linux / Termux / future AVF execution
  ├─ Web or Wasm execution
  └─ remote execution
          ↓
  Android task / WebView / Wayland / other presentation
```

This repository is both the **user documentation** and the **engineering knowledge base** for
Meldframe.

## Start here

If you are new to the project:

- **[What is Meldframe?](docs/INTRODUCTION.md)** — the project in plain terms.
- **[Features and current status](docs/FEATURES.md)** — what is implemented, verified, experimental or still planned.
- **[Using Meldframe](docs/USAGE.md)** — current development-build workflow and runtime setup.
- **[Capabilities](docs/CAPABILITIES.md)** — why Meldframe says Available, Experimental, Broken, Unknown, and more.
- **[Extensions and plugins](docs/EXTENSIONS.md)** — apps, runtime providers, compatibility layers and future third-party plugins.
- **[Architecture overview](docs/ARCHITECTURE.md)** — how execution, presentation, runtimes and windows fit together.

## The core idea

Meldframe does not assume:

```
application = Android package
window      = Activity
runtime     = local Android process
```

Instead it treats these as separate dimensions.

For example, **Code** can conceptually be:

```
code-server running in Linux
        +
Android WebView presentation
        +
one Meldframe window/task identity
```

A future Linux GUI application may instead use Wayland, while an Android application stays an
ordinary Android task. Users should interact with applications, not backend taxonomy.

## Current project highlights

The current development architecture already includes:

- a unified application catalogue and launch coordinator;
- backend-neutral window/task state;
- capability-aware planning;
- Android work-profile application identity;
- a runtime-provider model;
- Meldframe Terminal with xterm.js and pluggable transports;
- external Termux command/service integration;
- service supervision;
- code-server as a service-backed application;
- typed extension contributions;
- evidence-based WebAssembly capability probing.

Major areas such as full Wayland multi-window integration, AVF Linux providers, external extension
packaging, Browser/Playwright brokers and deeper Android system integration remain active roadmap
work. See [Features and current status](docs/FEATURES.md) before assuming a feature is production-ready.

## For users

### Applications

Meldframe aims to make Android apps, internal apps, service-backed Web apps and Linux applications
feel like members of the same desktop rather than separate subsystems.

### Capabilities

Device support is not represented by a single yes/no flag. Meldframe records whether a feature is
available, needs setup, is experimental, is known broken, is unsupported, or is simply not known yet.

Read [Capabilities](docs/CAPABILITIES.md) before troubleshooting hardware/OEM/runtime differences.

### Plugins

“Plugin” is the user-facing term for Meldframe extensions. A plugin may contribute an app, runtime,
terminal transport, capability probe, presentation backend, compatibility adapter or another typed
feature.

The external third-party package format is **not stable yet**; current built-in extensions are being
used to validate the API before it is frozen.

Read [Extensions and plugins](docs/EXTENSIONS.md).

## For extension authors and contributors

Start with:

- [Architecture overview](docs/ARCHITECTURE.md)
- [Extensions and plugins](docs/EXTENSIONS.md)
- [Capabilities](docs/CAPABILITIES.md)
- [Development principles](development/PRINCIPLES.md)
- [Architecture Decision Records](decisions/README.md)

One important rule:

> A logical extension is not automatically a separate APK or process.

Semantic boundaries are designed first. Deployment boundaries are chosen later from licensing,
security, crash isolation, update cadence and performance.

## Research and ideas

The wiki is also where we keep material that is useful but not yet an implementation commitment:

- [ChromeOS, Android and Meldframe](research/chromeos-android-meldframe.md)
- [Ideas backlog](ideas/README.md)
- [Primary source index](sources/README.md)

Research notes may describe experiments, alternatives and rejected directions. Their status is
explicit so they are not confused with product promises.

## Decisions and source of truth

The knowledge flow is:

```
source / experiment
       ↓
research note
       ↓
idea / proposal
       ↓
ADR / accepted decision
       ↓
implementation contract + tests
```

Document status vocabulary:

- `research` — evidence and analysis, no commitment;
- `idea` — deliberately speculative;
- `proposal` — concrete design under consideration;
- `accepted` — architectural decision accepted;
- `normative` — contributor rule;
- `obsolete` — retained for history.

The implementation repository, executable tests and current architecture contracts remain
authoritative for what the current build actually does. The wiki explains the product, teaches its
concepts, records research and preserves **why** decisions were made; it must not silently become a
second implementation truth.

## Naming

The canonical project and product name is **Meldframe**.

Planned project site: `meldframe.cmys.top`  
Android application ID: `top.cmys.meldframe`
