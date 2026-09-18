# What is Meldframe?

**Audience:** users, contributors, and anyone evaluating the project  
**Status:** user documentation

Meldframe is a **composable desktop runtime and shell for Android**.

It began as an Android desktop launcher, but its scope has grown beyond arranging Android apps in a
desktop-shaped UI. Meldframe is being designed so Android applications, Web applications, Linux
programs, service-backed tools and future Wasm/Wayland applications can appear in one desktop while
remaining free to use the execution and presentation backend that fits them best.

A short description is:

> Meldframe gives heterogeneous applications one desktop identity, while keeping their runtime,
> presentation, capabilities and compatibility layers replaceable.

## Why this exists

A conventional Android launcher usually assumes:

```
app = Android package
window = Activity
launch = Intent
```

That model breaks down when the desktop also needs to host:

- an Android application in a native task;
- a terminal whose shell is running in Termux or Linux;
- code-server running as a Linux service but displayed in an Android WebView;
- a Linux GUI application presented through Wayland;
- a Web/Wasm application;
- a remote application or service;
- an application compatibility layer such as an Electron or browser adapter.

Meldframe therefore separates several concepts that are often coupled:

```
application identity
execution backend
presentation backend
runtime
window/task state
capabilities
compatibility adapter
```

The shell can then recombine them into one user-visible application.

## Example: one application, multiple possible plans

A future “Code” entry could be backed by different implementations depending on the device:

```
Code
├─ Linux VS Code           + Wayland
├─ code-server             + Android WebView
├─ Electron-compatible app + Android WebView
└─ remote code-server      + local WebView
```

The user should still see **Code**, not an internal description of which backend happened to win.

## What Meldframe is not

Meldframe is not intended to be:

- a complete replacement Android OS;
- a Linux desktop stuffed into one Android window;
- a browser-first operating system;
- a mandatory Termux frontend;
- a single universal compatibility layer that pretends every program is identical.

Android remains the host platform. Meldframe prefers to use Android's own WindowManager/WM Shell,
SurfaceFlinger, input/IME, WebView/Chromium and virtualization mechanisms, then provide policy and
composition above them.

## Current development direction

The current architecture already includes a typed application model, unified application catalogue,
launch coordinator, window registry, capability graph, extension registry, runtime-provider model,
Terminal domain model, a Termux runtime path, a service supervisor, code-server integration and a
WebView/Wasm capability path.

Other areas — particularly full Wayland multi-window integration, AVF-backed Linux runtimes,
external extension packaging, Browser/Playwright brokers and deeper system integration — are active
research/development directions rather than finished product features.

See:

- [Features and status](FEATURES.md)
- [Using Meldframe](USAGE.md)
- [Capabilities](CAPABILITIES.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
