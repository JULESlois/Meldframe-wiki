# Desktop sessions and shell modes

**Audience:** users and contributors  
**Status:** implemented architecture; hardware coverage varies

Meldframe deliberately separates two jobs that Android launchers often combine:

1. providing a desktop workspace;
2. acting as the device's Home/launcher application.

This matters on phones, tablets and external displays. A user may want Meldframe on an attached
monitor while keeping the normal phone launcher unchanged.

## The three shell modes

| Mode | Desktop workspace | Launcher/Home duties | Intended use |
| --- | --- | --- | --- |
| `DESKTOP_ONLY` | yes | no | Recommended/default desktop behavior; especially secondary-display or manually launched sessions |
| `HYBRID` | yes | yes | Meldframe is Home and also presents a desktop |
| `FULL_HOME` | no | yes | Launcher-oriented behavior without assuming a desktop posture |

`DESKTOP_ONLY` is the conservative fallback. Merely tapping Meldframe does not imply that the user
wants to replace the device launcher.

These names describe **shell policy**, not execution backends. They do not tell you whether Linux,
Wayland, privileged Android task control or another capability is available.

## How Meldframe chooses a mode

The current implementation separates observation from policy:

```text
Android/platform observations
        ↓
SessionConditions
        ↓
DesktopSessionResolver
        ↓
SessionDecision
        ↓
DesktopSessionManager
```

The resolver uses the first applicable rule in this order:

1. explicit user override;
2. secondary display;
3. detected vendor desktop mode;
4. detected platform desktop-windowing posture;
5. launch as Android Home;
6. large-screen posture (`sw600dp`);
7. otherwise `DESKTOP_ONLY`.

A user override intentionally wins over automatic detection. Device detection is a starting point,
not an authority that should repeatedly undo a user's choice.

## Home is not the desktop signal

Older launcher designs often reduce the decision to:

```text
CATEGORY_HOME → draw desktop
```

Meldframe does not.

`CATEGORY_HOME` matters when Meldframe was actually launched as Home, but it does not override a
stronger signal such as a secondary display. This is required for the common desktop-mode shape:

```text
phone display
→ normal launcher

external display
→ Meldframe desktop
```

The architecture is therefore not tied to replacing the phone's launcher.

## Unknown is not false

Some Android desktop signals do not have a reliable public API. Meldframe represents uncertain
observations as unknown rather than inventing a negative result.

For example, generic Android multi-window state is not proof of desktop windowing: split-screen on a
phone can produce similar platform state. Treating it as a desktop signal would create false
positives.

This follows the same rule as the capability model:

```text
UNKNOWN ≠ UNSUPPORTED
```

When Meldframe cannot establish a desktop-specific condition, resolution should fall toward the
conservative mode rather than force desktop policy from weak evidence.

## Device profiles

Device-specific detection is isolated behind `DeviceProfile` / `DeviceProfileRegistry` instead of
scattering manufacturer checks throughout shell code.

Current built-in profile identifiers include:

- `wsa`;
- `samsung`;
- `xiaomi`;
- `lenovo`;
- `generic`.

The implementation also contains vendor desktop-mode detection work for Samsung and Motorola.

This list must not be read as a compatibility guarantee. The WSA profile has recorded device
verification. Samsung, Motorola, Xiaomi and Lenovo matching/detection paths exist in code, but the
project's current evidence does **not** establish broad hardware compatibility for those vendors.
See [Compatibility and verification](COMPATIBILITY.md).

## Live session state

`DesktopSessionManager` holds the active decision and reacts to relevant configuration changes.
User overrides are persisted through launcher preferences. Diagnostics record the selected session,
its trigger/reason, display information and active device profile.

This is useful when automatic selection appears wrong: first determine what Meldframe actually
observed and which rule won, rather than assuming the visible mode proves a particular hardware
capability.

## What shell mode does not enable

Selecting a desktop-oriented shell mode does **not** automatically grant:

- privileged Android task move/resize/close operations;
- root or Shizuku access;
- a Linux runtime;
- Wayland/XWayland presentation;
- GPU acceleration;
- work-profile support;
- OEM desktop APIs.

Those are independent capabilities and providers. Consult [Capabilities](CAPABILITIES.md),
[Android desktop integration](ANDROID_DESKTOP.md), [Runtimes](RUNTIMES.md), and
[Compatibility and verification](COMPATIBILITY.md).

## Current evidence boundary

The session manager, resolver, user override path and device-profile architecture are implemented.
The project has concrete WSA evidence, but representative physical-device validation remains
incomplete. In particular, the presence of a vendor profile or detector in source is not evidence
that the corresponding OEM desktop mode has been validated end to end.

For implementation details, the authoritative source remains `docs/SESSION_MODEL.md` and the
current code in `JULESlois/Hyperdroid-recovery`.
