# Code and service-backed applications

**Audience:** users, testers and extension authors  
**Status:** Code/code-server path implemented and verified on WSA; general service-app model remains under development

Code is Meldframe's first substantial example of an application whose **execution** and
**presentation** live in different environments.

```
code-server
  runs in Linux / Termux
        ↓
ServiceSupervisor
        ↓
Meldframe loopback gateway
        ↓
Android WebView
        ↓
one Meldframe Code window
```

The user interacts with Code as one application. The Linux service is an implementation detail, not
a second desktop.

## Why this matters

Running a full Linux Chromium instance just to display a local web application adds memory, GPU and
integration costs. A service-backed application can instead keep the backend where Linux tooling is
strong while using Android's WebView for presentation.

This pattern is not automatically better for every application. Meldframe should split an
application only when it buys compatibility, integration, resource use or maintainability.

## Launch behavior

Code is contributed as a typed `ServiceApp` extension. Its contribution describes the application,
service and path to open. Generic shell components handle the rest.

The planner selects Linux-service execution plus WebView presentation. The `ServiceSupervisor`
checks/starts the service and tracks health. A failed service should produce a meaningful launch
failure/state instead of a blank browser window.

When a just-started service is not ready yet, the current implementation can show “Starting Code…”
and retry until the health endpoint answers.

## Security boundary

The service is not exposed to the WebView as a bare `--auth none` endpoint.

The current implementation uses a per-service loopback token proxy with an ephemeral port and a
per-process session cookie held by that application's WebView. Requests without the expected session
are rejected. This narrows the local attack surface, but it should not be confused with a complete
sandbox.

The implementation documentation records one remaining design gap: the service's own listening port
still exists behind the proxy. Moving the backend side to a Unix socket is the intended way to close
that gap.

## Integrated window chrome

Code also demonstrates that Meldframe can integrate its window controls into a web application's own
title-bar region without forking the application.

The generic service-app host supports a default Meldframe frame and an integrated-chrome mode. Code's
integration script adds minimise, maximise/restore, close and drag behavior to VS Code's title-bar
area. If integration fails or the renderer/service fails, the host can fall back to normal Meldframe
chrome.

This approach is preferable to maintaining a permanent fork of code-server solely for window
appearance.

## Popup behavior

The service-backed window does not blindly navigate itself when a web application requests a new
window. The current popup policy admits absolute HTTP(S) targets and can route a foreground popup to
Meldframe's browser/UI chrome instead.

The real Termux code-server path was reverified on WSA on 2026-09-17: a test `window.open(...)`
created a second WebView target while the Code window remained on its own gateway URL.

This is a narrow verified behavior, not yet a general browser compatibility guarantee.

## Verified environment

The implementation repository records a complete WSA test path on 2026-09-17 using Termux User
Repository `code-server 4.137.0`. It verified service startup through Termux `RUN_COMMAND`, `/healthz`,
the workbench, WebSockets and extension-host connectivity.

Earlier WSA testing also verified the integrated title bar and the service-backed window path.

These versions/dates are evidence points. They should not be read as a promise that only that version
works or that every Android WebView/OEM environment behaves identically.

## What Code does not prove

Code is strong evidence for the service-backed architecture, but it does not prove that arbitrary
Electron or browser applications can be converted automatically.

Future `ElectronCompat`, Browser Broker and Playwright integration remain separate research/roadmap
work. Applications may depend on Chromium/Electron APIs, native modules, process topology or browser
features that WebView does not provide.

## For extension authors

The useful lesson is the shape, not Code-specific code:

```
AppDescriptor
+ ServiceSpec
+ presentation requirements
+ optional integrated chrome
= ServiceApp contribution
```

Keep application-specific startup knowledge in the extension/adapter. Keep service lifecycle,
health, gateway policy, window registry and WebView hosting generic.

## Related documentation

- [Installation and setup](INSTALLATION.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
- [Capabilities](CAPABILITIES.md)
- [Troubleshooting](TROUBLESHOOTING.md)
