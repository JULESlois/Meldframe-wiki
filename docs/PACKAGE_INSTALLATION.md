# Package Installation and AppImage

**Audience:** users, extension authors and contributors  
**Status:** proposal

Meldframe should eventually provide a package-installation layer for applications that are not
Android APKs.

**AppImage** is the proposed first reference format because it exercises nearly every subsystem that
Meldframe wants to unify:

```
FileRef
Package inspection
Runtime selection
Architecture compatibility
Installation transaction
Desktop metadata
MIME associations
Launcher integration
Execution planning
Wayland presentation
WindowRegistry
```

The important design rule is:

> AppImage support should validate generic package and desktop-integration APIs, not add AppImage
> special cases to Shell Core.

## User experience

The intended flow is:

```
download Foo.AppImage
        ↓
double-click in Explorer
        ↓
AppImage Installer
        ↓
inspect package
        ↓
choose / resolve runtime
        ↓
show compatibility and integrations
        ↓
install
        ↓
appears in Start/Search/Open With
        ↓
launch
        ↓
Meldframe window/task
```

## Generic package architecture

Do not define the Core API as `installAppImage()`.

Use a package abstraction:

```kotlin
interface PackageInstaller {
    suspend fun inspect(file: FileRef): PackageInspection
    suspend fun prepare(inspection: PackageInspection): InstallPlan
    suspend fun commit(plan: InstallPlan, grant: InstallGrant): InstallResult
}
```

AppImage is simply the first provider.

Future providers may include AppImage, `.deb`, Flatpak, portable tarballs, web-app packages and
Meldframe extension packages.

## Inspection

A useful AppImage inspection should determine:

- package architecture;
- embedded desktop entry;
- icon;
- executable/AppRun information;
- MIME associations;
- categories;
- expected runtime family;
- likely GUI/presentation requirements;
- whether extraction is required;
- obvious incompatibilities.

A result should distinguish:

```
compatible
compatible with setup
compatible through translation
unsupported
unknown
```

instead of reducing everything to a boolean.

## Architecture compatibility

AppImage is architecture-specific.

A package built for x86_64 should not be presented as natively runnable on an arm64 Android device.

The installer should ask the execution planner for candidates:

```
PackageInspector
→ requires linux/x86_64

ExecutionPlanner
├ native arm64 provider       incompatible
├ translated x86 provider     maybe available
└ remote x86 provider         available
```

The Installer should not hard-code Box64, FEX, QEMU or a specific remote solution. Those are
execution providers.

## libc and runtime compatibility

A normal AppImage generally expects a standard Linux userspace. Termux-native executables use
Android's bionic-based environment, so “Termux is installed” is not enough evidence that an AppImage
will run.

Preferred candidates are Linux environments providing the expected userspace, for example Debian
PRoot, root/chroot, AVF Debian guest or remote Linux.

The AppImage identity must not become tied to the runtime chosen at installation time.

## FUSE must be optional

The installer should not assume that FUSE is available in every Android/container environment.

Model the execution strategy explicitly:

```
FUSE_MOUNT
EXTRACT_AND_RUN
PERSISTENT_EXTRACT
```

A fallback extraction path is important for PRoot and other constrained runtimes.

## Reuse desktop-entry parsing

AppImage already carries Linux desktop metadata.

Do not invent AppImage-specific application metadata.

The path should be:

```
embedded .desktop
       ↓
DesktopEntryParser
       ↓
AppDescriptor
```

The same parser should later serve normal Linux `.desktop` discovery.

Relevant fields include `Name`, `Exec`, `Icon`, `TryExec`, `Path`, `Terminal`, `Actions`,
`MimeType`, `Categories` and `StartupWMClass`.

## Installation is a transaction

Installing an app affects more than one subsystem.

A proposed plan:

```kotlin
data class InstallPlan(
    val packageRef: PackageRef,
    val runtime: RuntimeRef,
    val executionStrategy: ExecutionStrategy,
    val appDescriptor: AppDescriptor,
    val associations: Set<MimeAssociation>,
    val requestedIntegrations: Set<DesktopIntegration>
)
```

The lifecycle should be:

```
inspect
   ↓
prepare plan
   ↓
show plan + permissions/integrations
   ↓
user approve
   ↓
commit
   ↓
publish app
```

If commit fails, rollback should remove partially extracted files, launcher registration,
associations, icon/cache state and installer-owned runtime state.

## Desktop integrations belong to portals

The installer must not directly edit Start-menu JSON, desktop databases or taskbar internals.

It requests desktop integration through:

```
LauncherPortal
AssociationPortal
NotificationPortal
FilePortal
```

Changing a default handler is a user-mediated request, not an installer entitlement.

## Portal-driven installation UI

An installer can present requested integrations before commit:

```
Foo Editor

Runtime:
  Debian

Execution:
  extracted AppImage

Requests:
  Handle text/plain
  Handle text/markdown
  Add launcher
  Create desktop shortcut

Optional:
  Make default text editor
```

## Installed application model

The result should publish a normal Meldframe application:

```
AppDescriptor
├ AppId
├ name
├ icon
├ categories
├ MIME handlers
├ execution candidates
├ runtime requirements
└ presentation requirements
```

The shell should not know that the source package happened to be an AppImage.

## Launch path

After installation:

```
Start / Search / Explorer
        ↓
AppId
        ↓
ApplicationCoordinator
        ↓
ExecutionPlanner
        ↓
RuntimeProvider
        ↓
Linux process
        ↓
Wayland/XWayland/Web-compatible presentation
        ↓
WindowRegistry
        ↓
Taskbar
```

## Uninstall

Uninstall must be symmetrical with install.

It should remove package-owned files, AppDescriptor registration, MIME associations, desktop
shortcuts, cached icons, installer-owned grants and optional persistent extraction.

It should not remove unrelated shared runtime packages or user data without explicit policy.

## Why AppImage Installer is a reference application

Terminal proves runtime/session abstractions.

Code proves service-backed split runtime.

AppImage Installer can prove package, file, association, portal and Linux-GUI integration at once.

A strong acceptance scenario is:

```
download AppImage
→ double-click
→ inspect
→ install
→ appears in Start
→ MIME handler appears
→ launch
→ independent Meldframe window
→ app calls xdg-open and gets Meldframe Browser
→ app calls FileChooser and gets Meldframe file portal
→ uninstall removes all owned integration
```

If this works without an AppImage-specific branch in Shell Core, the abstraction has earned its
place.
