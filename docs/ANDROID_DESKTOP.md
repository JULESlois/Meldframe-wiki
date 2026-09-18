# Android desktop integration and privileged task control

**Audience:** users, testers and contributors  
**Status:** living guide; checked against `Hyperdroid-recovery` main on 2026-09-18

Meldframe runs as an Android application, but a desktop shell needs information and controls that an ordinary APK does not automatically have. This page explains that boundary so that “Android support”, “root support” and “window control” are not treated as the same claim.

## The important distinction

Launching an Android application and controlling the device-wide Android task set are different operations.

An ordinary Meldframe installation can participate in normal Android application launching. It cannot, merely by being installed, enumerate and manipulate every task on the device. Android protects desktop-wide task observation and control behind privileged/signature-level boundaries.

The current architecture therefore separates:

```text
application launch
        ≠
desktop-wide task observation
        ≠
move / resize / close / maximize
```

A provider must prove the operations it can perform before Meldframe exposes them.

## Current provider model

The implementation uses a provider boundary rather than assuming one privilege mechanism:

```text
Android task provider
        ↓
full native snapshot
        ↓
reconciler
        ↓
WindowRegistry

shell command
        ↓
capability + router gate
        ↓
provider request
        ↓
later full snapshot confirms the result
```

An accepted command is **not** treated as proof that Android changed state. Meldframe waits for a later authoritative snapshot. This is intentional: privileged commands can time out, partially succeed, be rejected by an OEM build, or report misleading output.

## Without a privileged provider

If no provider has proved desktop-wide access, task observation/control/resize remain setup-dependent rather than silently pretending to work.

This is not equivalent to “Meldframe cannot run Android apps”. It means the shell cannot yet take authoritative control of the entire Android task model.

On the WSA development environment, attempts from Meldframe's ordinary app UID to use desktop-wide task inspection are rejected because the relevant Android permissions are privileged/signature permissions rather than ordinary runtime permissions.

## Magisk provider: experimental and opt-in

The first privileged provider adapter is `MagiskAndroidTaskBackend`.

Its current design is deliberately fail-closed:

- creating or probing the provider does not launch `su` or open a root prompt;
- only an explicit access request invokes MagiskSU;
- access is accepted only after uid 0 and a valid full task snapshot are proved;
- unknown device/build combinations do not inherit control commands merely because root exists;
- control capability is granted operation by operation;
- after every command attempt, another command is blocked until a fresh snapshot establishes actual Android state.

Root therefore means **a possible provider path**, not “all Meldframe desktop features are enabled”.

## What has actually been verified on WSA

For the exact WSA fingerprint recorded by the implementation project (`2407.40000.4.0`, Android 13), shell-level experiments established command semantics for:

- move/resize;
- maximize by resizing to observed maximum bounds;
- restore to previously confirmed normal bounds;
- close for a safely identified single-task root.

The same experiments did **not** establish a safe general `ACTIVATE`, `MINIMIZE` or fullscreen contract.

More importantly, the in-app Magisk provider is not yet a successful end-to-end product path on that test installation: MagiskSU currently denies Meldframe's application UID. A developer succeeding with `adb shell su` is not evidence that Meldframe itself has been granted root, because those are different callers.

Consequently the correct current claim is:

> The provider architecture and WSA command semantics exist; a successful in-app privileged control round trip and product setup UI remain open work.

## Device profiles are evidence filters, not OEM promises

Meldframe has device-profile infrastructure and detection for environments including WSA and several OEM desktop-mode families. That does not mean every named OEM is verified.

The current implementation explicitly distinguishes a verified WSA profile from manufacturer/device matching that still needs hardware evidence. A profile should narrow or override behavior only when the device/build evidence justifies it.

Avoid conclusions such as:

```text
Samsung detected → all DeX controls supported
root detected    → all task controls supported
Android 13       → WSA command dialect supported
```

The relevant unit of evidence may be an exact build fingerprint and a proved operation set.

## Why capabilities are operation-specific

A provider that can observe tasks may still be unable to resize them. A provider that can close one kind of task may not safely activate arbitrary tasks.

Conceptually:

```text
OBSERVE
MOVE
RESIZE
CLOSE
MAXIMIZE
RESTORE
ACTIVATE
MINIMIZE
```

are separate facts.

This is why Meldframe does not expose one broad `root = true` or `taskControl = true` switch as the architectural truth.

## Root, Shizuku and system integration

Magisk is the first adapter, not the permanent definition of privileged Android integration.

Shizuku and privileged/system-service approaches remain candidate peer providers behind the same contracts. They should be documented as implemented only after code and device evidence exist.

A future provider can therefore change *how* access is obtained without changing `WindowRegistry`, the taskbar or application identity.

## What testers should record

For privileged Android desktop testing, record:

1. device model and Android version;
2. exact build fingerprint;
3. Meldframe commit/build;
4. provider name;
5. provider capability state;
6. operations the provider says it proved;
7. whether authorization was explicitly granted;
8. the full-snapshot result before and after a command;
9. the command attempted and its exact failure if any.

Do not report a successful adb/root-shell command as a successful Meldframe provider test unless the operation was executed through the application/provider path being evaluated.

## What is not claimed yet

This page does not claim that:

- root is required for ordinary Meldframe use;
- root automatically enables Android desktop control;
- Magisk task control is a finished user-facing feature;
- activate/minimize/fullscreen are verified on WSA;
- Samsung, Motorola, Xiaomi or Lenovo desktop modes are broadly verified;
- Shizuku or a system-service provider is already shipped;
- a command returning success is sufficient to mutate Meldframe's window state.

For the broader evidence matrix, see [Compatibility and verification status](COMPATIBILITY.md). For the meaning of capability states, see [Capabilities](CAPABILITIES.md).