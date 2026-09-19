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

## What works today

The current implementation has an end-to-end Code path, not merely a design for one. With a working
Termux runtime and code-server installation, Meldframe can start the service through Termux
`RUN_COMMAND`, wait for its health endpoint, open the VS Code workbench in a Meldframe-owned WebView,
and carry WebSocket traffic used by the workbench and extension host.

The implementation repository records real-device-path acceptance on WSA on 2026-09-17 with Termux
User Repository `code-server 4.137.0`. It also verified keyboard input, integrated title-bar controls
and foreground popup routing. These are evidence points, not a compatibility promise for every
Android WebView, OEM image or future code-server release.

| Behavior | Current status |
| --- | --- |
| Code appears as a built-in Meldframe application | Implemented |
| Start code-server through the Termux runtime | Implemented; WSA verified |
| Wait/retry while a newly started service becomes healthy | Implemented |
| Workbench, WebSockets and extension-host connection | WSA verified |
| Meldframe controls integrated into VS Code's title bar | Implemented; WSA verified |
| Foreground HTTP(S) popup routed to Meldframe browser chrome | Narrow path verified |
| Arbitrary web/Electron application conversion | Not supported as a general feature |
| Unix-socket-only backend isolation | Planned; current backend still has its own loopback port |
| Browser Broker / Playwright compatibility layer | Research/roadmap |

## Before launching Code

Code depends on the current external-Termux runtime path. Complete the Termux setup in
[Installation and setup](INSTALLATION.md) first: Termux must allow external commands and Meldframe
must have `com.termux.permission.RUN_COMMAND`. A package being installed is not sufficient evidence
that the runtime is usable; Meldframe's runtime probe must succeed.

code-server is a separate Linux-side dependency. The version verified by the implementation
repository is `4.137.0` from the Termux User Repository. The wiki intentionally does not turn that
single tested version into a hard version pin. If installation packaging changes, the implementation
repository remains the source of truth.

A healthy launch should progress through these distinct stages:

```
Termux runtime READY
      ↓
code-server process started or already observed
      ↓
/healthz answers
      ↓
gateway session created
      ↓
Code WebView opens
      ↓
workbench connects, including WebSockets
```

Keeping these stages separate matters when diagnosing failures. A visible WebView does not prove the
Linux service is healthy, and a listening service does not prove the WebView/gateway path works.

## Launch behavior

Code is contributed as a typed `ServiceApp` extension. Its contribution describes the application,
service and path to open. Generic shell components handle the rest.

The planner selects Linux-service execution plus WebView presentation. `ServiceSupervisor` starts or
observes the service and tracks health. A just-started service can be reported as `Running` while the
window shows “Starting Code…” and retries until the endpoint answers. A service that remains
unhealthy is reported as such rather than being treated as ready merely because a process launch was
attempted.

This distinction is deliberate: process existence, TCP reachability and application readiness are
not interchangeable states.

## Security boundary

The service is not exposed to its application WebView as a bare `--auth none` endpoint.

The current implementation uses one `LoopbackTokenProxy` per service. The proxy gets an ephemeral
port and a per-process session cookie held by that application's WebView; requests without the
expected session are rejected. Android cleartext HTTP is permitted by Meldframe's network security
configuration for loopback only, rather than globally.

This reduces the local attack surface but is **not a complete sandbox**. The implementation still
records one important gap: code-server's own listening port exists behind the proxy. Moving the
backend side to a Unix socket is the intended way to remove that additional loopback listener.
Therefore “gateway protected” should not be read as “the backend has no independently reachable
socket”.

## Integrated window chrome

Code demonstrates that Meldframe can integrate its window controls into a web application's own
title-bar region without maintaining a code-server fork.

`ServiceAppChrome.Integrated` names an extension asset. The generic service-app host injects the
integration script, which adds minimise, maximise/restore, close and drag behavior to VS Code's
title-bar area. The shell frame is hidden before the first layout so a successful takeover does not
shift the page after rendering.

The integration is deliberately fail-soft. If title-bar integration times out, the renderer is lost
or the service fails, the host can restore normal Meldframe chrome. The current implementation also
coalesces Android mouse-hover handling because WSA WebView behavior does not line up cleanly with the
page's CSS hover feature reporting.

## Popup behavior

A service-backed window does not blindly replace itself when the hosted application requests a new
window. The current popup policy admits absolute HTTP(S) targets and can route a foreground popup to
Meldframe's browser/UI chrome instead.

The real Termux code-server path was reverified on WSA on 2026-09-17: a test `window.open(...)`
created a second WebView target while the Code window remained on its own gateway URL. Background
popups are still rejected by the current policy.

This is a narrow verified behavior, not a general browser compatibility guarantee. Do not infer that
all VS Code extensions, OAuth flows or arbitrary popup patterns have been tested.

## Diagnosing Code failures

Classify a failure by the earliest stage that is wrong rather than treating every blank or delayed
window as a WebView problem:

1. **Runtime unavailable or needs setup.** Verify the Termux provider first. Missing
   `RUN_COMMAND` permission and `allow-external-apps` are runtime setup problems, not Code problems.
2. **Service cannot start.** Preserve the supervisor/runtime error. The current path depends on
   Termux command execution; on API 26+ the implementation uses `startForegroundService` for
   Termux's `RunCommandService`, including after a cold/force-stopped Termux state.
3. **Service starts but health never becomes ready.** Treat `/healthz` failure as a service problem.
   Do not infer readiness from a process or port alone.
4. **Health is good but the window cannot load.** Investigate the gateway/session and WebView path.
   A direct backend URL is not the intended application boundary.
5. **Workbench loads but interactive features fail.** Separate ordinary HTTP rendering from
   WebSocket/extension-host behavior and from Android input/IME behavior.
6. **Only integrated controls fail.** The service may still be healthy. Normal Meldframe chrome is
   the fallback path; title-bar injection is a presentation integration layer.
7. **Only a popup fails.** Check the popup policy before blaming Code itself. Current support is
   intentionally narrower than arbitrary browser popup semantics.

See [Troubleshooting](TROUBLESHOOTING.md) for runtime-wide diagnostics.

## What Code does not prove

Code is strong evidence for the service-backed architecture, but it does not prove that arbitrary
Electron or browser applications can be converted automatically. Applications may depend on
Chromium/Electron APIs, native modules, process topology, DRM, browser security features or other
facilities that Android WebView does not provide.

Future `ElectronCompat`, Browser Broker and Playwright integration remain separate research/roadmap
work. They should not be described as Code features until the implementation repository contains a
working consumer and corresponding verification.

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
health, gateway policy, window registry and WebView hosting generic. A new service-backed extension
should also document separately what has only been unit-tested, what has been exercised against a
real service, and what has been verified on a target Android environment.

The current extension architecture is built-in only. `ServiceApp` being a real typed contribution
does **not** imply that third parties can already package and install external Meldframe plugins; see
[Extensions and plugins](EXTENSIONS.md) for that boundary.

## Related documentation

- [Installation and setup](INSTALLATION.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Architecture overview](ARCHITECTURE.md)
- [Capabilities](CAPABILITIES.md)
- [Troubleshooting](TROUBLESHOOTING.md)
