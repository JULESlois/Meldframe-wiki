# Ideas

**Status:** idea

This directory is for useful possibilities that are deliberately **not roadmap commitments**.

## Current idea clusters

### Runtime and guest integration

- Meldframe Runtime companion derived from the proven parts of the Termux ecosystem.
- A versioned Meldframe Guest Protocol shared by local containers, AVF guests, SSH runtimes and WSL.
- Progressive runtime installation: base PTY/exec → Linux rootfs → GUI/Wayland stack.
- Runtime planning that includes isolation/trust, not only compatibility and speed.

### Web and browser

- Browser Broker allowing Linux programs to use Android-native WebView/Chromium.
- Chromium CLI compatibility shim for `xdg-open`, `chromium --app` and automation workflows.
- Playwright/Puppeteer bridge to Android browser sessions.
- Dedicated Chromium/content-shell host only if WebView/Chrome APIs prove insufficient.
- WebAssembly/WASI extension runtime with semantic host capabilities.

### Presentation and windows

- Wayland per-toplevel integration rather than embedding a complete Linux desktop.
- Progressive frame transport from shared-memory copy to dma-buf/AHardwareBuffer.
- Unified Inspector adapters for Android, WebView, Wayland, Wasm and Terminal targets.
- Personality packages that can replace shell surfaces without forking task/window semantics.

### Desktop services and packages

- Meldframe Portal Framework for files, associations, launchers, wallpaper, notifications, shortcuts and other desktop services.
- `xdg-desktop-portal-meldframe` so unmodified Linux applications can use Meldframe-native desktop services.
- AppImage Installer as the first PackageInstaller workload, with extraction fallback when FUSE is unavailable.
- Generic package framework that can later admit `.deb`, Flatpak, portable bundles and Meldframe extension packages.
- User-activation tokens and scoped PortalGrant objects for sensitive/persistent desktop operations.

### Compatibility

- code-server as a first-class application adapter rather than "a page in a browser".
- ElectronCompat: Linux Node backend + Android WebView renderer for compatible apps.
- Service-backed compatibility profiles that can be expressed mostly as manifests.

## Rule

Move an item out of this directory only when there is enough evidence to write either a concrete
proposal or an ADR. Keeping speculative ideas cheap is intentional.
