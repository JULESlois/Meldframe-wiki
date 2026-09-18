# Source Index

**Status:** research

Prefer primary documentation and source code. Record the exact subsystem or claim a source supports
rather than adding links without context.

## ChromeOS

- ChromiumOS containers and VMs overview — Crostini components, lifecycle and host/guest integration:  
  https://www.chromium.org/chromium-os/developer-library/guides/containers/containers-and-vms/
- Crostini developer guide — component responsibilities and development details:  
  https://www.chromium.org/chromium-os/developer-library/guides/containers/crostini-developer-guide/
- Chromium App Service source/docs — unified application publishers/consumers and instance model:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/components/services/app_service/
- Exo — ChromeOS Wayland server used by ARC/Crostini integration:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/components/exo/README.md
- Sommelier — guest Wayland/X11 proxy and buffer strategies:  
  https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/vm_tools/sommelier/README.md
- Lacros design/history — browser/OS boundary and the cost of physical/release separation:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/docs/lacros.md
- ChromeOS System Web Apps source/docs:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/chrome/browser/ash/system_web_apps/

## Android

- Android Virtualization Framework use cases — including the Linux development environment:  
  https://source.android.com/docs/core/virtualization/usecases
- VirtualizationService / VM lifecycle:  
  https://source.android.com/docs/core/virtualization/virtualization-service
- Android desktop windowing architecture:  
  https://source.android.com/docs/core/display/desktop-windowing
- Android desktop UX / multitasking guidance:  
  https://developer.android.com/design/ui/desktop/guides/system/multi-task
- crosvm source and documentation:  
  https://chromium.googlesource.com/crosvm/crosvm/

## Termux

- Termux execution environment:  
  https://github.com/termux/termux-packages/wiki/Termux-execution-environment
- Building packages / custom package-name implications:  
  https://github.com/termux/termux-packages/wiki/Building-packages
- RUN_COMMAND integration:  
  https://github.com/termux/termux-app/wiki/RUN_COMMAND-Intent

## Desktop portals and Linux integration

- xdg-desktop-portal documentation — semantic desktop services for sandboxed apps:  
  https://flatpak.github.io/xdg-desktop-portal/docs/
- DynamicLauncher — user-mediated persistent launcher installation:  
  https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.DynamicLauncher.html
- Documents portal — scoped document access and grants:  
  https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Documents.html
- GlobalShortcuts portal — application/session-scoped shortcut registration:  
  https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.GlobalShortcuts.html
- Writing an xdg-desktop-portal backend — desktop-specific backend contract:  
  https://flatpak.github.io/xdg-desktop-portal/docs/writing-a-new-backend.html
- xdg-activation — compositor-mediated user activation tokens:  
  https://wayland.app/protocols/xdg-activation-v1
- freedesktop MIME application associations:  
  https://specifications.freedesktop.org/mime-apps/latest-single/
- Desktop Entry specification:  
  https://specifications.freedesktop.org/desktop-entry/latest/

## AppImage

- AppImage architecture — runtime plus filesystem image/AppRun model:  
  https://docs.appimage.org/reference/architecture.html
- AppImage FUSE troubleshooting and extraction fallback:  
  https://docs.appimage.org/user-guide/troubleshooting/fuse.html
- AppImage desktop integration metadata:  
  https://docs.appimage.org/reference/desktop-integration.html

## Android desktop-service APIs

- RoleManager — user-mediated Android roles/default-app requests:  
  https://developer.android.com/reference/android/app/role/RoleManager
- WallpaperManager — Android wallpaper provider mechanism:  
  https://developer.android.com/reference/android/app/WallpaperManager
- Background activity launch restrictions — host-side activation constraints:  
  https://developer.android.com/guide/components/activities/background-starts

## Windows compatibility / Winlator

- Winlator upstream repository — Android Windows compatibility stack and integration reference:  
  https://github.com/brunodev85/winlator
- Winlator application source repository — current app/core integration reference:  
  https://github.com/brunodev85/winlator-app
- Wine project — Win32 compatibility implementation and upstream Wayland/WoW64 work:  
  https://www.winehq.org/
- Box64 — x86_64 userspace translation on ARM64 and Wine integration:  
  https://github.com/ptitSeb/box64

These are implementation references for a future low-priority compatibility extension, not evidence
that Windows support is already part of Meldframe.

## Research hygiene

For each future research note:

1. capture primary sources first;
2. record the relevant revision/date when behavior is fast-moving;
3. distinguish current facts from Meldframe proposals;
4. attach experiments or code references when a claim can be tested;
5. mark obsolete conclusions instead of silently rewriting history.
