# Desktop portals, AppImage and native-feeling application integration

**Status:** research  
**Last reviewed:** 2026-09-19

## Research question

What does Meldframe need to expose so Android, Web, Wasm and Linux applications behave like native
members of one desktop rather than merely being displayed by it?

The research points toward a dedicated **Portal Framework** plus a generic **Package Framework**.

## External precedents

### xdg-desktop-portal

The freedesktop portal architecture is the closest direct precedent.

It provides desktop services to sandboxed applications through semantic interfaces rather than
giving applications unrestricted access to the desktop environment implementation.

Relevant interfaces include FileChooser/Documents, OpenURI, Wallpaper, Notification,
GlobalShortcuts, Background, DynamicLauncher and FileTransfer.

The key lesson is not D-Bus itself. It is the separation:

```
application
→ stable semantic portal
→ permission / user mediation
→ desktop-specific backend
```

Meldframe needs the same semantic boundary across more runtime types.

### Request and Session lifecycles

Portal operations are often asynchronous or long-lived.

User-mediated requests benefit from cancelable request objects; persistent resources such as
shortcuts/background sessions benefit from explicit session ownership.

This aligns with Meldframe's existing rule that lifecycle must come from the subsystem that owns it,
not from inference.

### DynamicLauncher

DynamicLauncher is particularly relevant to package installation.

Its prepare/approve/install shape suggests that persistent shell integration should be transactional
and user-mediated rather than “write a launcher file and hope”.

That maps naturally to AppImage installation:

```
inspect
→ prepare install plan
→ show integrations
→ user approve
→ commit launcher/associations/files
```

### XDG activation

Desktop focus and application activation should be tied to user intent.

Wayland's xdg-activation token model is a useful reference for a Meldframe
`UserActivationToken`: a short-lived proof that a request follows real input such as a click,
shortcut or notification action.

Android's background-activity-start restrictions point in the same direction: foreground activation
should not be an unrestricted plugin API.

## Android implications

Meldframe can provide its own desktop defaults and associations, but it cannot treat Android's system
defaults as arbitrary writable state.

Android roles such as Home or Browser are user-mediated. The safe abstraction is therefore:

```
registerHandler(...)
requestDefault(...)
```

not:

```
setDefaultSilently(...)
```

Likewise, Android-specific facilities such as WallpaperManager belong behind a portal provider rather
than in application-facing APIs.

## File access as capability forwarding

Cross-runtime file handling is a major reason to use a portal.

A raw Linux path is not an Android URI. An Android Content URI is not a Debian pathname.

The useful abstraction is:

```
resource
→ FileRef
→ scoped grant
→ target-runtime representation
```

This lets Meldframe forward access rather than globally exposing storage.

## Linux-native compatibility through xdg-desktop-portal-meldframe

The most valuable long-term compatibility idea is to implement a desktop-specific XDG portal backend.

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
```

That would allow existing Linux applications to use Meldframe desktop services without a
Meldframe-specific SDK.

Possible mappings:

| XDG portal | Meldframe |
| --- | --- |
| FileChooser | FilePortal / Explorer picker |
| OpenURI | OpenPortal / BrowserProvider |
| Notification | NotificationPortal |
| Wallpaper | WallpaperPortal |
| DynamicLauncher | LauncherPortal |
| GlobalShortcuts | ShortcutPortal |
| FileTransfer | FileRef + scoped transfer grant |

Wayland gives the app a native-feeling window. Portal compatibility gives it native-feeling desktop
services.

## AppImage as the first package workload

AppImage is useful because it is neither an Android package nor a simple archive from Meldframe's
point of view.

The installer has to reason about package format, architecture, Linux userspace expectations, FUSE
availability, extraction fallback, desktop metadata, MIME handlers, runtime selection and
presentation availability.

That makes it a strong validation workload for the planner and portal model.

## AppImage runtime constraints

### Architecture

AppImage packages are architecture-specific. An x86_64 AppImage does not become native simply because
the Android device can run Meldframe.

Architecture translation belongs to execution providers, not to the installer.

### libc / Linux userspace

Standard Linux AppImages generally expect a Linux/glibc-style environment. A Termux-native
bionic-based userspace is a different target.

Therefore AppImage should normally be planned against a suitable Linux runtime rather than treated as
a Termux-native package format.

### FUSE

AppImage's normal runtime path uses a mounted filesystem image, but FUSE is not guaranteed in Android
or containerized environments.

Extraction must be a first-class strategy, not a hidden emergency workaround.

## Generic package framework

The research suggests this decomposition:

```
PackageRef
   ↓
PackageInspector
   ↓
PackageInspection
   ↓
ExecutionPlanner
   ↓
InstallPlan
   ↓
PackageInstallerProvider
   ↓
Portal-backed desktop integration
   ↓
Installed AppDescriptor
```

This lets future package formats reuse the same shell integration.

## Security and UX consequences

Portal design implies several concrete rules:

- capability and permission stay separate;
- permissions can be scoped to one resource/session;
- sensitive operations may require current user activation;
- becoming the default handler requires user confirmation;
- persistent launcher/background/global-shortcut state should be inspectable and revocable;
- Web/Wasm/Linux callers receive semantic APIs, not raw Android host power;
- package installation is transactional.

## Proposed validation sequence

1. Open/Association/Launcher portals with unit tests.
2. AppImage inspection and plan generation.
3. Install an AppImage by extraction into a known Linux runtime.
4. Publish the resulting AppDescriptor and MIME handlers.
5. Launch through the normal planner.
6. Integrate a Linux FileChooser/OpenURI through XDG portal compatibility.
7. Uninstall and prove that all installer-owned state is removed.

## Primary sources

- xdg-desktop-portal documentation: https://flatpak.github.io/xdg-desktop-portal/docs/
- Wallpaper portal: https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.impl.portal.Wallpaper.html
- Documents portal: https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Documents.html
- DynamicLauncher portal: https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.DynamicLauncher.html
- GlobalShortcuts portal: https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.GlobalShortcuts.html
- Writing a portal backend: https://flatpak.github.io/xdg-desktop-portal/docs/writing-a-new-backend.html
- xdg-activation: https://wayland.app/protocols/xdg-activation-v1
- Android background activity launch restrictions: https://developer.android.com/guide/components/activities/background-starts
- Android RoleManager: https://developer.android.com/reference/android/app/role/RoleManager
- Android WallpaperManager: https://developer.android.com/reference/android/app/WallpaperManager
- freedesktop MIME applications specification: https://specifications.freedesktop.org/mime-apps/latest-single/
- Desktop Entry specification: https://specifications.freedesktop.org/desktop-entry/latest/
- AppImage architecture: https://docs.appimage.org/reference/architecture.html
- AppImage FUSE fallback: https://docs.appimage.org/user-guide/troubleshooting/fuse.html
- AppImage desktop integration: https://docs.appimage.org/reference/desktop-integration.html
- Termux execution environment: https://github.com/termux/termux-packages/wiki/Termux-execution-environment
