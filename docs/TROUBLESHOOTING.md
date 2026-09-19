# Troubleshooting

**Audience:** users and testers  
**Status:** development-build documentation

Meldframe deliberately distinguishes “not configured”, “temporarily unhealthy”, “not yet proven” and
“unsupported”. Start with the reported capability/runtime state rather than guessing from an app
being installed or a UI entry being visible.

## Read the state first

| State | First interpretation |
| --- | --- |
| `Available` | the relevant probe has positive working evidence |
| `AvailableWithSetup` | a required package, permission or setup step is missing |
| `Experimental` | evidence exists, but the path is not considered stable |
| `Broken` | Meldframe expected it to work but observed a failure |
| `Unsupported` | positive evidence says this path cannot work here |
| `Unknown` | Meldframe does not yet have enough evidence |

`Unknown` is not `Unsupported`.

## Termux is installed, but Meldframe says the runtime is not ready

Package presence is intentionally insufficient. There are two independent setup gates and then a
real execution probe.

First, check that Termux allows external applications in `~/.termux/termux.properties`:

```properties
allow-external-apps=true
```

Reload the setting with:

```sh
termux-reload-settings
```

Second, Meldframe must hold `com.termux.permission.RUN_COMMAND`. The implementation repository's WSA
development setup used:

```sh
adb shell pm grant top.cmys.meldframe com.termux.permission.RUN_COMMAND
```

Treat this as the currently recorded test setup, not a promise that ADB is the eventual end-user
permission flow.

Finally, the provider proves readiness by running `uname -a` through Termux's actual `RUN_COMMAND`
service and receiving stdout/stderr/exit status. Use the provider's reason to distinguish the cases:

- it asks for `RUN_COMMAND` → the permission gate is missing;
- it says Termux refused the command and mentions `allow-external-apps` → fix/reload the Termux
  property;
- both are configured but `uname -a` still fails → this is an execution/result-path failure and is
  worth reporting with the exact reason.

Do not “fix” diagnostics by forcing the capability to Available unless you are deliberately testing
the override path.

## A registered Linux application is missing from Start

Linux application **discovery and Start listing are implemented**, but they are narrower than a full
Linux desktop integration. Diagnose discovery before launch.

Current registration stores a container name and a directory path that the Android process can
actually read. Meldframe scans that registered tree for freedesktop `.desktop` entries. A path that
exists only inside a PRoot guest namespace is not automatically visible to Android; arbitrary
in-guest scanning still needs a future runtime execution path.

Check these separately:

1. Is the container registered, and is its registered directory still readable from Android?
2. Does the directory contain the expected `.desktop` entry?
3. Does the entry intentionally hide itself with `Hidden`, `NoDisplay`, `OnlyShowIn` or
   `NotShowIn` semantics?
4. Does the catalogue contain the Linux application even if Start search does not?
5. If two containers contain the same desktop id, check the full namespaced identity rather than
   assuming they are duplicates: Linux app identity is `container + desktopId`.

The implementation has been verified with real WPS Office and Cylheim desktop entries, including
localized WPS names on a `zh_CN` WSA device. That is evidence for the parser/listing path, not a
promise that every desktop file or Android storage layout works.

A generic icon is currently expected. Resolving a `.desktop` `Icon` value normally requires access to
the guest's icon themes, and that integration is not implemented yet.

## A Linux application appears in Start but does not launch

This is currently an **implementation boundary, not evidence that discovery failed**.

Meldframe can discover registered Linux `.desktop` applications and list them in Start, but there is
no production `AppId.Linux` GUI launch backend yet. Selecting such an entry therefore must not be
interpreted as equivalent to launching a native Android or service-backed application.

The M3 research PoC has separately proved that a real Linux GUI application can render through a
nested compositor and reach the RFB wire. It has **not** turned that experiment into the Start launch
path, production Wayland/XWayland presentation, or per-window Linux integration. Do not work around
the missing backend by marking Wayland or XWayland capabilities as available.

See [Linux applications](LINUX_APPS.md) for the current discovery/execution/presentation boundary.

## Terminal opens but cannot connect

The current PoC can use a ttyd loopback transport. A TCP listener existing on the configured port is
only startup evidence; the live WebSocket/ttyd protocol still has to succeed.

Check separately:

1. whether the selected runtime is healthy;
2. whether ttyd/service startup succeeded;
3. whether the expected loopback endpoint is listening;
4. whether the Terminal reports a transport/session failure after connecting.

Do not conflate a ttyd failure with “xterm.js is broken” or “Linux is unsupported”. They are different
layers.

## Terminal disconnects after being backgrounded

This is a known limitation of the current ttyd PoC, not a solved reconnect feature.

On WSA, a backgrounded/sleeping Android side stopped answering the WebSocket long enough for ttyd to
close the connection and terminate its shell. Meldframe then correctly reported the session as
disconnected, but ttyd had already destroyed the process.

Until a persistent runtime-agent/native-PTY path lands, use a persistent layer such as tmux when that
fits the test, or expect the ttyd-attached shell to be disposable. A future reconnect UI cannot by
itself resurrect a process that the transport already killed.

## Code opens a starting/error state instead of the workbench

That is preferable to a blank WebView. Code is service-backed, so separate these questions:

1. Can the selected RuntimeProvider execute commands?
2. Did code-server start?
3. Does its health endpoint answer?
4. Did the Meldframe gateway start?
5. Can the WebView reach the gateway?

A service that has just started may need time to become healthy. The current host can remain in a
“Starting Code…” state and retry rather than treating the first refused connection as permanent.

## A localhost Web app works in a browser but not in Meldframe

Loopback networking still crosses Android/WebView security policy. The implementation explicitly
permits cleartext HTTP only for loopback service use because Android otherwise rejects it by default.

Do not broaden that into a global cleartext-network exception when debugging. If the target is not a
Meldframe-owned loopback service, investigate the URL/origin/policy separately.

## A popup or external page behaves differently inside Code

Code is an application host, not a general browser tab. Its popup policy intentionally handles new
windows separately so an application cannot casually replace its host window.

The current verified path accepts foreground absolute HTTP(S) popup targets and can route them to
Meldframe browser/UI chrome. Other popup schemes/background behavior should not be assumed supported.

## Wasm reports `AvailableWithSetup`

A Wasm feature is probed by behavior rather than browser-version guessing. It is possible for base
WebAssembly and SIMD to work while threads do not.

For example, the WSA verification on 2026-09-17 found working Wasm and SIMD/atomic instructions, but
no `SharedArrayBuffer`/cross-origin isolation, so Wasm Threads correctly remained
`AvailableWithSetup`. WASI similarly requires an actual WASI host; a WebView merely supporting Wasm
is not enough.

## Desktop/window behavior differs by device

Do not immediately add an OEM-name conditional.

Meldframe's intended order is:

```
probe actual capability
→ apply a known data-driven device-profile correction if necessary
→ apply explicit user override last
```

A useful report includes Android version, device/OEM, Meldframe capability state and reason, runtime
health, and the exact operation that failed.

## What to include in a bug report

Prefer evidence over conclusions. Include:

- Meldframe build/commit if known;
- Android version and device/environment;
- capability state **and reason**;
- selected runtime/provider and health state;
- application/extension involved;
- minimal reproduction steps;
- relevant Meldframe diagnostics/log excerpt;
- whether the failure is reproducible after restarting the affected runtime/service.

Avoid posting credentials, service tokens, private file paths or unrelated installed-app lists.

## Before assuming a roadmap feature is broken

Check [Features and current status](FEATURES.md). AVF runtime support, production Wayland
multi-window, external third-party plugin packaging, Browser/Playwright broker and several privileged
system integrations are roadmap/research work. A missing implementation is not a runtime regression.

## Related documentation

- [Installation and setup](INSTALLATION.md)
- [Using Meldframe](USAGE.md)
- [Capabilities](CAPABILITIES.md)
- [Linux applications](LINUX_APPS.md)
- [Terminal](TERMINAL.md)
- [Code](CODE.md)
