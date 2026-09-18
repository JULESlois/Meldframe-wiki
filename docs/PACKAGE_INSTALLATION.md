# Meldframe Installer and Package Installation

**Audience:** users, extension authors and contributors  
**Status:** proposal

Meldframe already has an internal **Installer** application inherited from the predecessor. Today that
application is effectively a managed Web App installer: it accepts a name and URL, stores app
metadata and icon information, optionally creates a desktop shortcut, refreshes the application
surface and launches the result through the shell.

That makes it a useful product concept to preserve.

The proposed direction is:

> **Do not create separate Web App Installer, AppImage Installer and Windows Installer applications.
> Evolve the existing Installer into one provider-driven Meldframe Installer.**

The Installer is the user-facing shell. Format/runtime-specific logic belongs to install providers.

## Product model

The user should be able to give Meldframe a URL or file and get one consistent install experience.

Examples:

```
https://example.app
Foo.AppImage
foo.desktop
portable-linux-bundle
Foo.exe
Setup.exe
Package.msi
future Meldframe extension package
```

The Installer first identifies what was supplied and then delegates inspection/planning/execution to
the matching provider.

```
URL / FileRef
     ↓
Meldframe Installer
     ↓
InstallSourceResolver
     ↓
PackageInspector / InstallProvider
     ↓
PackageInspection
     ↓
InstallPlanner
     ↓
InstallPlan
     ↓
user review / approval
     ↓
transactional commit
     ↓
AppDescriptor + Portal integrations
```

The shell sees an installed application, not a package-format special case.

## Preserve the existing Web App installer as Provider #1

The current installer already proves a basic workflow:

```
name + URL + icon + shortcut option
        ↓
managed WebApp record
        ↓
application metadata
        ↓
Start/Desktop refresh
```

That behavior should migrate behind a `WebAppInstallProvider`, rather than be thrown away.

A richer provider can later inspect:

- Web App Manifest;
- title/name;
- favicon/icons;
- start URL;
- scope;
- display mode;
- service-worker/PWA metadata;
- permissions/capabilities Meldframe wants to expose;
- whether the user wants a standalone app entry.

The existing manual form remains a valid fallback when automatic metadata is unavailable.

## Universal installer architecture

Avoid a single interface that assumes every install source behaves like a file archive.

A useful split is:

```kotlin
interface InstallSourceResolver {
    suspend fun resolve(source: InstallSource): List<InstallerCandidate>
}

interface InstallProvider {
    val id: InstallProviderId

    suspend fun inspect(source: InstallSource): PackageInspection
    suspend fun prepare(inspection: PackageInspection): InstallPlan
    suspend fun commit(plan: InstallPlan, grant: InstallGrant): InstallResult
    suspend fun rollback(transaction: InstallTransaction)
}
```

`InstallSource` can represent:

```
WebUrl
FileRef
ExistingRuntimePath
RemotePackageRef
ExtensionPackageRef
```

This prevents the Web App path from being awkwardly forced into a fake local package model.

## Common inspection model

Each provider should return common shell-facing facts where possible:

```
PackageInspection
├ kind
├ identity candidate
├ display name
├ icon candidates
├ version
├ architecture
├ runtime requirements
├ presentation requirements
├ capability requirements
├ desktop metadata
├ MIME / URI handlers
├ requested integrations
├ security/trust notes
└ compatibility result
```

Provider-specific details remain attached as opaque provider data.

Compatibility should be richer than yes/no:

```
compatible
compatible with setup
compatible through translation
experimental
unsupported
unknown
```

## Installer UX

The UI can stay conceptually simple even while providers become more capable.

A generic flow:

```
Choose / drop / open source
        ↓
Inspecting…
        ↓
Application summary
        ↓
Runtime / compatibility summary
        ↓
Desktop integrations and permissions
        ↓
Install
        ↓
progress / diagnostics
        ↓
Launch
```

For simple Web Apps the form may still be:

```
Name
URL
Icon
Create desktop shortcut
Install
```

For AppImage:

```
Foo Editor
Linux / aarch64
Runtime: Debian
Execution: Extract and run
Handles: text/plain, text/markdown
[Install]
```

For a Windows executable:

```
Foo Editor
Windows x64
Compatibility: Winlator provider
Environment: Auto
Presentation: compatibility display path
[Install]
```

Advanced backend details should be hidden unless the user asks for them.

## Linux install providers

Linux support should be provider-driven rather than one giant “Linux package installer”.

### AppImage

AppImage remains a strong first Linux workload because it exercises:

- package architecture;
- Linux userspace compatibility;
- FUSE vs extraction;
- embedded `.desktop` metadata;
- icons;
- MIME associations;
- runtime selection;
- GUI presentation.

The execution strategy should explicitly support:

```
FUSE_MOUNT
EXTRACT_AND_RUN
PERSISTENT_EXTRACT
```

The Installer should ask the planner which Linux runtime can satisfy the package rather than hard-code
Termux, PRoot, chroot or AVF.

### Existing .desktop applications

A `.desktop` entry may not need “installation” in the package-manager sense.

The provider can register an application already present inside a RuntimeInstance:

```
runtime path / .desktop entry
        ↓
DesktopEntryParser
        ↓
AppDescriptor
        ↓
LauncherPortal
```

This is useful for applications installed through apt/pacman/other tools inside a guest.

### .deb / distro packages

These can be added later, but the generic Installer should not directly mutate distro package
databases itself.

Instead:

```
.deb / package request
→ Linux package provider
→ selected RuntimeProvider
→ distro package manager / transactional runtime service
```

Shared-runtime package changes have broader consequences than installing a self-contained AppImage,
so they need explicit ownership/uninstall policy.

### Flatpak and other formats

Possible future providers, not commitments.

They should only be added when the target runtime can honestly supply the services they need.

## Windows install provider through WinCompat

Windows support should reuse the proposed Winlator-derived Windows compatibility engine.

The Installer does not manage Wine/Box64/DXVK itself.

```
.exe / .msi
     ↓
WindowsInstallProvider
     ↓
WindowsCompatibilityProvider
     ↓
Winlator-derived Core
```

### Portable executable

The simplest Windows acceptance case:

```
Foo.exe
→ inspect PE metadata / architecture
→ create or select CompatibilityEnvironment
→ register executable
→ publish AppDescriptor
```

### Installer executable or MSI

A richer path:

```
Setup.exe / Package.msi
→ inspect
→ create/select CompatibilityEnvironment
→ run installer
→ discover resulting shortcuts/app records
→ let user choose what to publish
→ publish normal AppDescriptors
```

Meldframe should not expose Winlator's launcher UI as the installed-app surface.

See [Windows Compatibility through a Winlator-derived Core](WINDOWS_COMPATIBILITY.md).

## Reuse desktop-entry and app discovery models

Package formats should not invent parallel application metadata.

Examples:

```
AppImage embedded .desktop
Linux runtime .desktop
Windows discovered shortcut
Web App manifest
        │
        ▼
provider-specific adapter
        │
        ▼
AppDescriptor
```

This is where the Installer hands off to the rest of Meldframe.

## Installation is a transaction

Installing an app may affect multiple systems.

A conceptual plan:

```kotlin
data class InstallPlan(
    val provider: InstallProviderId,
    val source: InstallSource,
    val appCandidates: List<AppDescriptor>,
    val runtimePlan: RuntimeInstallPlan?,
    val requestedIntegrations: Set<DesktopIntegration>,
    val providerPayload: ProviderInstallPayload
)
```

The lifecycle should be:

```
inspect
   ↓
prepare
   ↓
user review
   ↓
commit
   ↓
publish apps / integrations
```

If commit fails, rollback should remove all installer-owned partial state where the provider can do so
safely.

A package manager changing a shared distro may need a different rollback policy from a self-contained
AppImage or Web App; the transaction model should represent that rather than pretending all providers
are identical.

## Desktop integration belongs to Portals

The Installer must not directly mutate Start, taskbar, defaults or desktop implementation data.

It should request:

```
LauncherPortal
AssociationPortal
FilePortal
NotificationPortal
Shortcut/Desktop integration
```

Examples:

```
publish application
register text/plain handler
request default editor
create desktop shortcut
notify install completion
```

A provider contributes facts; Portal policy decides what persistent desktop integration is allowed.

## Application identity is independent of install technology

Do not encode the package/runtime environment into the identity merely because it was used to install
the app.

Examples:

```
Managed Web App
AppImage
Windows EXE through Winlator
Linux app discovered from .desktop
```

all become normal application descriptors after installation.

The original package source remains provenance, not the primary UI identity.

## Uninstall is provider-owned but shell-coordinated

The Meldframe Installer/Apps UI should provide one uninstall entry point.

The provider implements actual removal:

```
WebApp provider
→ remove managed app record

AppImage provider
→ remove extracted/self-contained files

Linux distro provider
→ invoke package manager when ownership is clear

Windows provider
→ invoke uninstall metadata or remove owned compatibility environment/app
```

The shell then removes provider-owned:

- AppDescriptor registrations;
- associations;
- shortcuts;
- cached icons;
- installer grants;
- installation records.

User data and shared runtime components require explicit policy.

## Reference workloads

The universal Installer gives Meldframe several excellent architecture tests.

### Web

```
URL
→ inspect manifest
→ install
→ Start
→ WebView
```

### AppImage

```
AppImage
→ inspect architecture/desktop metadata
→ select Linux runtime
→ install/extract
→ Start
→ Wayland
```

### Windows

```
portable EXE
→ inspect PE
→ WinCompat provider
→ install/register
→ Start
→ compatibility presentation
```

If all three work through one user-facing Installer without format checks leaking into Shell Core, the
package architecture is doing useful work.

## Implementation sequence

Because Linux/Wayland is the current priority, a sensible order is:

1. Refactor the existing managed-WebApp Installer behind `WebAppInstallProvider` without changing
   product behavior.
2. Add generic `InstallSource`, inspection and transaction models.
3. Add AppImage as the first Linux package provider.
4. Reuse Linux `.desktop` discovery for applications already installed in runtimes.
5. Complete Portal-backed launcher/association integration.
6. Only then add a low-priority `WindowsInstallProvider` backed by the Winlator-derived core.
7. Add distro package formats only when ownership/rollback semantics are understood.

The Installer itself can therefore evolve now while Windows compatibility remains intentionally
low priority.
