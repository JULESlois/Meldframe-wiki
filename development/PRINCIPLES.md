# Meldframe Development Principles

**Status:** normative

These rules exist to keep the project composable while it grows.

## 1. Identity is not implementation

`AppId`, window identity and user-visible task identity must not encode the currently selected
runtime, transport or presentation backend unless that fact is genuinely part of identity.

## 2. Execution and presentation are orthogonal

A Linux process may be presented by Wayland, WebView or another adapter. A WebView application may
have a Linux, Android, Wasm or remote backend. Do not collapse these axes for convenience.

## 3. Consume facts; do not infer lifecycle

A successful launch request does not mean a window exists. A successful exec does not mean a GUI
toplevel exists. Registry state comes from the subsystem that owns the actual lifecycle event.

## 4. Capabilities, not brand conditionals

Prefer probes, capability states and explicit device profiles over scattered checks for Samsung,
Xiaomi, WSA or particular Android versions.

## 5. Logical extension != process boundary

Design a typed contribution boundary first. Deploy it in-process, in another APK/process, in a Linux
service or in a Wasm sandbox only when a concrete reason requires that boundary.

## 6. Protocol before provider lock-in

Termux, PRoot, AVF, chroot and SSH are providers. Cross-runtime semantics belong in a stable Meldframe
protocol where possible.

## 7. Use the host's strongest mechanism

Android already has WindowManager/WM Shell, SurfaceFlinger, WebView/Chromium, input/IME, media and
virtualization primitives. Meldframe should orchestrate them rather than recreate them.

## 8. Split applications only when it buys something

"Linux backend + Android frontend" is valuable when it improves GPU compatibility, memory usage,
integration or UX. Do not split an application merely because the architecture permits it.

## 9. Security boundaries are explicit

PRoot is not a security sandbox. Localhost is not an authorization mechanism. Privileged Web/Wasm
host APIs require permissions, origin/trust checks and narrow semantic contracts.

## 10. Optimize after semantic correctness

For Linux graphics, prove toplevel ownership, lifecycle, input and resize before insisting on
zero-copy dma-buf. For transports, preserve process/session semantics before micro-optimizing IPC.

## 11. Every abstraction needs a workload

Validate architecture with real clients:

```
RuntimeProvider       → Terminal
ServiceSupervisor     → code-server
BrowserProvider       → Playwright / Web apps
FileProvider          → Explorer
WaylandPresentation   → a real Linux GUI app
Wasm capability       → a real extension/module
DebugAdapter          → Inspector
```

An abstraction without a workload remains provisional.

## 12. Preserve one source of truth

Research and ideas live in this wiki. Accepted implementation contracts must be reflected in the main
repository and enforced by code/tests. Do not let wiki prose silently override executable behavior.

## 13. Prefer graceful fallback

Experimental root, WM Shell, Wayland, GPU and runtime paths must fail back to a known working mode.
A capability being unavailable must not corrupt the shell's application/window model.

## 14. Keep compatibility at the edge

Legacy encodings, OEM quirks, ttyd limitations, browser shims and protocol translations belong in
adapters/providers. Core models should express Meldframe semantics, not the accidental shape of one
backend.
