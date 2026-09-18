# Meldframe Portal Framework

**Audience:** users, extension authors and contributors  
**Status:** proposal

Meldframe should expose desktop capabilities through a **typed Portal Framework** rather than
through ad-hoc helpers or direct access to Android internals.

The Portal Framework is the proposed boundary between applications and desktop services such as
opening files and URLs, choosing files, registering launchers and file handlers, requesting default
associations, setting wallpaper, notifications, global shortcuts, background activity, clipboard,
window actions, runtime services and browser integration.

The central rule is:

> Applications ask Meldframe for a desktop service. They do not modify shell state or call host
> implementation details directly.

## Why a portal layer is needed

Without a portal boundary, each application tends to grow a private integration path:

```
Code special API
Terminal special API
Explorer special API
AppImage Installer special API
```

That recreates the coupling Meldframe is trying to remove.

The Portal Framework instead provides one semantic API shared by built-in Kotlin apps, Web apps,
Wasm/WASI apps, Linux guests, remote apps and CLI tools. The transport can differ; the meaning does
not.

## Architecture

```
                      Application
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
 Kotlin/Core API       JS/Web API         Guest/CLI API
       │                   │                   │
       └──────────── Meldframe Portal ─────────┘
                           │
                      PortalBroker
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   PermissionBroker  CapabilityGraph   GrantStore
          │                │                │
          └────────────────┼────────────────┘
                           │
                    Portal Provider
         ┌─────────────────┼──────────────────┐
         │                 │                  │
      Android            Shell             Runtime
```

The host implementation can change without changing application-facing semantics.

For example:

```
WallpaperPortal
        ↓
Android provider
        ↓
WallpaperManager
```

A Web app should never need an Android object reference to set a wallpaper.

## Strongly typed portals

Do not build one giant `Portal.doEverything()`.

The proposed family is:

| Portal | Responsibility | Typical policy |
| --- | --- | --- |
| `OpenPortal` | Open URI/FileRef, Open With | default handler or chooser |
| `FilePortal` | Pick/open/save/grant files | scoped resource grants |
| `LauncherPortal` | Register/remove app launcher or shortcut | prompt/approval for persistent integration |
| `AssociationPortal` | MIME handlers and default-app requests | registration may be cheap; changing default requires confirmation |
| `WallpaperPortal` | Desktop/lock-screen wallpaper request | preview and/or confirmation |
| `NotificationPortal` | Show/update/remove notifications | permission + policy |
| `ShortcutPortal` | Global shortcuts | session + explicit approval |
| `BackgroundPortal` | Background/autostart request | explicit grant |
| `WindowPortal` | Create/focus/fullscreen/progress/badge | activation policy |
| `ClipboardPortal` | Clipboard read/write | reads are more sensitive |
| `RuntimePortal` | Exec/service/runtime operations | privileged |
| `BrowserPortal` | Open browser/session/automation | automation is separately privileged |
| `ThemePortal` | Theme/accent/personality metadata | mostly read-only |
| `SystemPortal` | System/session/power operations | strongly privileged |

The first implementation should not start with the most privileged portals. A low-risk set is a
better model validator:

```
Open
Associations
Launcher
Files
Wallpaper
Notifications
```

## Capability, permission and policy are different questions

A portal request should only execute after three independent checks.

```
Capability
    ↓
Can this environment technically do it?

Permission
    ↓
Is this application allowed to ask?

Policy / context
    ↓
Is this particular call allowed now?
```

Example:

```
wallpaper provider works
→ capability AVAILABLE

application has desktop.wallpaper.set
→ permission GRANTED

request has valid user activation / preview accepted
→ policy ALLOW
```

This is deliberately different from treating the capability graph as a security model.

## Portal caller identity

Every request needs a stable caller identity.

A proposed model:

```kotlin
data class PortalCaller(
    val appId: AppId,
    val extensionId: ExtensionId?,
    val runtime: RuntimeRef?,
    val trust: CallerTrust
)
```

The broker should derive caller identity from the connection/session where possible rather than trust
an arbitrary string supplied by the caller.

## Requests and sessions

Desktop operations are not all ordinary RPC calls.

A file chooser may wait for user input. A global shortcut may stay alive for hours.

Use two lifecycle shapes:

```
PortalRequest
├ requestId
├ caller
├ parentWindow
├ state
├ cancel()
└ result

PortalSession
├ sessionId
├ owner
├ state
├ capabilities
└ close()
```

This keeps long-running ownership explicit and allows cancellation/cleanup when the client
disconnects.

## User activation tokens

Applications should not be able to steal focus or silently trigger sensitive desktop integration
from the background.

Meldframe should mint a short-lived `UserActivationToken` from real user interaction:

```
mouse/touch click
keyboard activation
launcher selection
notification click
global shortcut
explicit portal confirmation
```

Operations such as `window.focus`, `app.launch`, `requestDefault`, `launcher.install` and
opening external UI may require a valid token.

The token is evidence of recent user intent, not a general permission.

## Grants

A simple `Map<AppId, Set<Permission>>` is not sufficient for file access and other scoped desktop
operations.

Proposed model:

```kotlin
data class PortalGrant(
    val id: GrantId,
    val subject: AppId,
    val permission: PortalPermission,
    val scope: GrantScope,
    val expiresAt: Instant?
)
```

Useful scopes include:

```
Once
Session
Resource(FileRef)
Workspace
Persistent
```

Suggested policy states:

```
ALLOW
PROMPT
DENY
PRIVILEGED_ONLY
```

## FileRef becomes the resource-security boundary

The File Portal should not expose raw Android paths or raw guest filesystem paths as the universal
API.

The preferred shape is:

```
raw Android Uri / Linux path / remote path
                ↓
             Broker
                ↓
           brokered FileRef
                ↓
             FileGrant
```

Cross-runtime transfer then becomes capability forwarding rather than shared-filesystem assumptions.

Example:

```
Android Explorer
    ↓ Open in Code
ContentUri
    ↓
FileRef
    ↓
FileBridge
    ↓
runtime-scoped grant
    ↓
Debian-visible path or FD
```

## Defaults and associations

Applications should be able to **register** handlers but only **request** becoming the default.

The API should therefore look conceptually like:

```
registerHandler(mime, app)
requestDefault(mime, app, activationToken)
```

not:

```
setDefaultSilently(...)
```

Meldframe can own its internal desktop defaults independently of Android system defaults.

Example:

```
text/plain
→ Code

image/png
→ Photos

application/vnd.appimage
→ AppImage Installer
```

Android system roles such as Browser or Home remain subject to Android's own user-mediated APIs.

## Transport bindings

The same portal semantics should be reachable from several environments.

### Kotlin / built-in apps

```kotlin
portal.files.open(...)
portal.wallpaper.requestSet(...)
```

### Web apps

```javascript
await meldframe.files.open(...)
await meldframe.notifications.show(...)
```

### Wasm/WASI

Expose semantic WIT interfaces, not raw host internals.

### Linux guest

Use Meldframe Guest Protocol and, where possible, standard compatibility APIs.

### CLI

A future CLI can map onto the same portals:

```bash
meldframe open report.pdf
meldframe wallpaper set image.png
meldframe default request text/plain code
meldframe notify "Build completed"
```

## Versioning

Do not use one global SDK integer for every portal.

Version each portal independently:

```
portal.files@3
portal.open@2
portal.wallpaper@1
portal.notifications@1
```

An extension manifest can then declare:

```yaml
requires:
  portals:
    files: ">=2"
    open: ">=1"

optional:
  wallpaper: ">=1"
```

## XDG compatibility

A long-term target is an `xdg-desktop-portal-meldframe` backend:

```
Linux application
      ↓
xdg-desktop-portal
      ↓
xdg-desktop-portal-meldframe
      ↓
Meldframe Guest Protocol
      ↓
PortalBroker
      ↓
Android / Meldframe shell
```

This is strategically important because existing Linux applications could gain Meldframe-native
integration without being modified specifically for Meldframe.

Examples:

```
FileChooser  → Meldframe file picker / Explorer
OpenURI      → BrowserProvider
Notification → Meldframe notifications
Wallpaper    → WallpaperPortal
DynamicLauncher → LauncherPortal
GlobalShortcuts → ShortcutPortal
```

Wayland and portals solve different halves of Linux desktop integration:

```
Wayland
= presentation / input / window integration

Portal
= desktop service integration
```

Both are needed for a Linux app to feel native to the Meldframe desktop.

## Implementation order

Proposed sequence:

1. Core contracts: `PortalCaller`, `PortalRequest`, `PortalSession`, `PortalGrant`,
   `UserActivationToken`.
2. `OpenPortal`, `AssociationPortal`, `LauncherPortal`.
3. `FilePortal` integrated with `FileRef` and scoped grants.
4. Wallpaper/Notification/Shortcut portals with a small Web/Wasm demo.
5. JS/Wasm bindings.
6. Guest Protocol transport.
7. `xdg-desktop-portal-meldframe`.
8. External-extension permission UI and persistent GrantStore.

This document is a proposal. None of these portal APIs should be treated as stable until they have
been exercised by real applications.
